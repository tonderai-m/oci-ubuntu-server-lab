# 04 OCI Ubuntu Server Troubleshooting

## Goal

Document common errors and fixes when setting up an Ubuntu server on Oracle Cloud Infrastructure.

This note focuses on:

```text
SSH problems
Route table problems
Subnet problems
Internet Gateway problems
Security rule problems
Cloud-init problems
Docker and Nginx checks
```

Project setup belongs in:

```text
notes/01-project-setup-ssh-cloud-init.md
```

Networking setup belongs in:

```text
notes/02-networking-vcn-subnet-gateway.md
```

Compute instance creation belongs in:

```text
notes/03-compute-instance-first-login.md
```

---

# 1. Fast Diagnosis Flow

When SSH does not work, diagnose in this order:

```text
1. Is the instance running?
2. Does the instance have a public IPv4 address?
3. Is the subnet public?
4. Does the route table have 0.0.0.0/0 → Internet Gateway?
5. Is the Internet Gateway enabled?
6. Does the Security List or NSG allow TCP 22?
7. Is the correct username being used?
8. Is the private key being used, not the .pub key?
9. Are key permissions correct?
```

The most important split:

```text
Operation timed out = network problem
Permission denied publickey = SSH key or username problem
```

---

# 2. Test Port 22 Before SSH

Before running SSH, test the network path:

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

If `nc` times out, SSH will also time out.

---

# 3. Error: Operation Timed Out

Example:

```text
ssh: connect to host <PUBLIC_IP> port 22: Operation timed out
```

## Meaning

Your Mac could not reach port `22` on the instance.

This usually means:

```text
The network path is blocked.
The SSH key has not even been checked yet.
```

## Common Causes

```text
No route rule to Internet Gateway
Internet Gateway missing or disabled
Subnet is private
Instance has no public IP
Security List blocks TCP 22
Network Security Group blocks TCP 22
Local network blocks outbound SSH
Instance is not fully booted
```

## Fix Checklist

Check instance:

```text
Compute → Instances → Instance Details
```

Confirm:

```text
State: Running
Public IPv4 address: Present
Subnet: public-subnet-01
```

Check subnet:

```text
Networking → VCN → Subnets → public-subnet-01
```

Confirm:

```text
Subnet Access: Public Subnet
```

Check Internet Gateway:

```text
VCN → Internet Gateways
```

Confirm:

```text
Internet Gateway exists
State: Available
Enabled: Yes
```

Check route table:

```text
VCN → Route Tables → Route table attached to public-subnet-01
```

Required route:

```text
Destination: 0.0.0.0/0
Target Type: Internet Gateway
Target: VCN Internet Gateway
```

Check security rule:

```text
Source: YOUR_PUBLIC_IP/32 or 0.0.0.0/0
Protocol: TCP
Destination Port: 22
Stateless: No
```

Then test again:

```bash
nc -vz <PUBLIC_IP> 22
```

---

# 4. Error: Permission Denied Publickey

Example:

```text
Permission denied (publickey).
```

## Meaning

The network path works.

SSH reached the instance, but authentication failed.

## Common Causes

```text
Using the public key instead of the private key
Wrong username
Wrong private key
Public key was not attached to the instance
Private key permissions are too open
Instance was created with a different SSH key
```

## Fix

Use the private key:

```bash
ssh -i ~/.ssh/oci_ubuntu_server_lab ubuntu@<PUBLIC_IP>
```

Do not use:

```bash
ssh -i ~/.ssh/oci_ubuntu_server_lab.pub ubuntu@<PUBLIC_IP>
```

Fix private key permissions:

```bash
chmod 600 ~/.ssh/oci_ubuntu_server_lab
```

Use correct default username:

```text
Ubuntu image: ubuntu
Oracle Linux image: opc
```

Try Ubuntu:

```bash
ssh -i ~/.ssh/oci_ubuntu_server_lab ubuntu@<PUBLIC_IP>
```

Try Oracle Linux only if the image is Oracle Linux:

```bash
ssh -i ~/.ssh/oci_ubuntu_server_lab opc@<PUBLIC_IP>
```

---

# 5. Error: No Route Rules Displayed

Example:

```text
Route Rules
No items to display
```

## Meaning

The route table has no route rules.

A public subnet without this route:

```text
0.0.0.0/0 → Internet Gateway
```

cannot route internet traffic properly.

## Fix

Go to:

```text
VCN → Route Tables → Route table attached to public-subnet-01
```

Add:

```text
Destination CIDR Block: 0.0.0.0/0
Target Type: Internet Gateway
Target: VCN Internet Gateway
Description: Public subnet internet route
```

Then test:

```bash
nc -vz <PUBLIC_IP> 22
```

---

# 6. Error: Route Table Requires Private IP Target

Example:

```text
Rules in the route table must use private IP as a target.
Or the route table can be empty.
```

## Meaning

OCI is treating that route table as a special route table for internal routing, not as a normal public subnet route table.

For a normal public Ubuntu server, do not choose:

```text
Private IP
```

as the target.

## Fix

Create a new route table:

```text
VCN → Route Tables → Create Route Table
```

Use:

```text
Name: public-subnet-route-table
```

Add route:

```text
Destination CIDR Block: 0.0.0.0/0
Target Type: Internet Gateway
Target: VCN Internet Gateway
```

Then attach it to the subnet:

```text
VCN → Subnets → public-subnet-01 → Edit → Route Table
```

Choose:

```text
public-subnet-route-table
```

Then test again:

```bash
nc -vz <PUBLIC_IP> 22
```

---

# 7. Error: Connection Refused

Example:

```text
ssh: connect to host <PUBLIC_IP> port 22: Connection refused
```

## Meaning

The instance is reachable, but nothing is accepting SSH connections on port `22`.

## Common Causes

```text
SSH service is not running
Instance is still booting
Operating system firewall is blocking SSH
Wrong port
Bad image initialization
```

## Fix

Wait a few minutes after instance creation.

If you have console access, check:

```bash
sudo systemctl status ssh
sudo systemctl restart ssh
```

On Ubuntu, the SSH service is usually:

```text
ssh
```

not:

```text
sshd
```

Check listening ports:

```bash
sudo ss -tulpn | grep :22
```

---

# 8. Error: Bad Permissions on Private Key

Example:

```text
WARNING: UNPROTECTED PRIVATE KEY FILE!
Permissions 0644 for 'oci_ubuntu_server_lab' are too open.
```

## Meaning

SSH refuses to use the private key because other users could read it.

## Fix

Run:

```bash
chmod 600 ~/.ssh/oci_ubuntu_server_lab
```

Then reconnect:

```bash
ssh -i ~/.ssh/oci_ubuntu_server_lab ubuntu@<PUBLIC_IP>
```

---

# 9. Error: Host Key Verification Failed

Example:

```text
Host key verification failed.
```

or:

```text
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

## Meaning

Your Mac has a saved SSH fingerprint for that IP or hostname, but the server changed.

This can happen if:

```text
You recreated the instance
The public IP was reused
The OS was reinstalled
The host key changed
```

## Fix

Remove the old known host entry:

```bash
ssh-keygen -R <PUBLIC_IP>
```

Then connect again:

```bash
ssh -i ~/.ssh/oci_ubuntu_server_lab ubuntu@<PUBLIC_IP>
```

When prompted, type:

```text
yes
```

---

# 10. Error: Public IP Exists but SSH Still Times Out

## Meaning

A public IP alone is not enough.

The full path must exist:

```text
Public IP
+ Public Subnet
+ Internet Gateway
+ Route Table
+ TCP 22 Security Rule
```

## Check

Instance details:

```text
Public IPv4 address: Present
Subnet: public-subnet-01
```

Subnet details:

```text
Subnet Access: Public Subnet
Route Table: public-subnet-route-table
```

Route table:

```text
0.0.0.0/0 → Internet Gateway
```

Security:

```text
TCP 22 allowed
```

Then test:

```bash
nc -vz <PUBLIC_IP> 22
```

---

# 11. Error: Instance Has No Public IP

## Meaning

The instance cannot be directly reached from the internet.

## Fix

Check instance VNIC:

```text
Compute → Instance → Attached VNICs → Primary VNIC
```

Look for:

```text
Public IPv4 address
```

If missing, possible fixes:

```text
Assign an ephemeral public IP to the VNIC
Recreate the instance in a public subnet with public IP enabled
Use a bastion host
Use a private network/VPN setup
```

For this beginner lab, the easiest path is:

```text
Public subnet + automatically assigned public IPv4
```

---

# 12. Error: Wrong Username

## Symptoms

```text
Permission denied (publickey).
```

## Correct Usernames

Use:

```text
Ubuntu: ubuntu
Oracle Linux: opc
CentOS: opc or centos
Debian: debian
```

For this lab:

```bash
ssh -i ~/.ssh/oci_ubuntu_server_lab ubuntu@<PUBLIC_IP>
```

---

# 13. Error: Used `.pub` File by Mistake

Wrong:

```bash
ssh -i ~/.ssh/oci_ubuntu_server_lab.pub ubuntu@<PUBLIC_IP>
```

Correct:

```bash
ssh -i ~/.ssh/oci_ubuntu_server_lab ubuntu@<PUBLIC_IP>
```

Explanation:

```text
The .pub file is the public key.
It goes into Oracle Cloud.
The private key has no extension by default and stays on the Mac.
```

---

# 14. Check Cloud-Init

After login, check:

```bash
cloud-init status --long
```

If cloud-init failed, inspect logs:

```bash
sudo tail -n 100 /var/log/cloud-init.log
sudo tail -n 100 /var/log/cloud-init-output.log
```

Search for errors:

```bash
sudo grep -i error /var/log/cloud-init.log
sudo grep -i failed /var/log/cloud-init.log
```

Common cloud-init problems:

```text
Bad YAML indentation
Missing #cloud-config first line
Broken package repository
Network not ready during first boot
Docker repository setup failed
```

---

# 15. Check Package Updates

Run:

```bash
sudo apt update
```

If it fails, check internet from the instance:

```bash
ping -c 4 8.8.8.8
curl -I https://ubuntu.com
```

If outbound internet fails, check:

```text
Route table
Internet Gateway
Egress security rule
DNS resolution
```

---

# 16. Check Docker

Check Docker version:

```bash
docker --version
```

Check Docker Compose:

```bash
docker compose version
```

Check Docker service:

```bash
systemctl status docker
```

If Docker is installed but permission is denied:

```bash
sudo usermod -aG docker ubuntu
exit
```

Then log back in and test:

```bash
docker run hello-world
```

If Docker is not installed, install manually:

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
```

Then follow the Docker install steps for Ubuntu.

---

# 17. Check Nginx

Check Nginx status:

```bash
systemctl status nginx
```

Start Nginx:

```bash
sudo systemctl start nginx
```

Enable Nginx on boot:

```bash
sudo systemctl enable nginx
```

Test locally on the instance:

```bash
curl -I http://localhost
```

To reach Nginx from the internet, security rules must allow:

```text
TCP 80
```

For HTTPS later:

```text
TCP 443
```

---

# 18. Web Page Not Loading

## Symptom

SSH works, but browser cannot open:

```text
http://<PUBLIC_IP>
```

## Causes

```text
Nginx not running
TCP 80 not open in Security List or NSG
OS firewall blocking port 80
Route table issue
Wrong public IP
```

## Fix

Check Nginx:

```bash
systemctl status nginx
```

Check local response:

```bash
curl -I http://localhost
```

Open OCI ingress rule:

```text
Source CIDR: 0.0.0.0/0
Protocol: TCP
Destination Port: 80
```

For HTTPS:

```text
Source CIDR: 0.0.0.0/0
Protocol: TCP
Destination Port: 443
```

---

# 19. PostgreSQL Public Access Warning

Do not expose PostgreSQL directly to the public internet.

Avoid:

```text
Source: 0.0.0.0/0
Port: 5432
```

Better options:

```text
SSH tunnel
Private subnet
VPN
Bastion host
Local-only Docker network
```

Example SSH tunnel pattern:

```bash
ssh -L 5432:localhost:5432 ubuntu@<PUBLIC_IP>
```

---

# 20. Useful Diagnostic Commands on Mac

Check SSH port:

```bash
nc -vz <PUBLIC_IP> 22
```

Verbose SSH:

```bash
ssh -vvv -i ~/.ssh/oci_ubuntu_server_lab ubuntu@<PUBLIC_IP>
```

Remove old known host:

```bash
ssh-keygen -R <PUBLIC_IP>
```

Check SSH config:

```bash
cat ~/.ssh/config
```

List keys:

```bash
ls -la ~/.ssh/
```

Check private key permissions:

```bash
ls -l ~/.ssh/oci_ubuntu_server_lab
```

---

# 21. Useful Diagnostic Commands on Server

System info:

```bash
whoami
hostname
uname -a
cat /etc/os-release
```

Disk and memory:

```bash
df -h
free -h
```

Network:

```bash
ip addr
ip route
```

Services:

```bash
systemctl status ssh
systemctl status nginx
systemctl status docker
```

Listening ports:

```bash
sudo ss -tulpn
```

Cloud-init:

```bash
cloud-init status --long
```

Logs:

```bash
sudo journalctl -xe
sudo journalctl -u ssh
sudo journalctl -u nginx
sudo journalctl -u docker
```

---

# 22. Quick Fix Table

| Problem | Likely Cause | Fix |
|---|---|---|
| `Operation timed out` | Network blocked | Check route table, gateway, public IP, TCP 22 |
| `Permission denied (publickey)` | Wrong key/user | Use private key and `ubuntu` |
| `No route rules displayed` | Missing internet route | Add `0.0.0.0/0 → Internet Gateway` |
| Private IP target error | Wrong route table context | Create new public route table |
| `Connection refused` | SSH service issue | Check `systemctl status ssh` |
| Bad key permissions | Key file too open | Run `chmod 600` |
| Browser cannot load Nginx | Port 80 blocked | Add TCP 80 ingress rule |
| Docker permission denied | User not in docker group | Run `sudo usermod -aG docker ubuntu` |
| `apt update` fails | Outbound internet/DNS issue | Check route, egress, DNS |
| Host key warning | Recreated instance/IP reused | Run `ssh-keygen -R <PUBLIC_IP>` |

---

# 23. Final Mental Model

For SSH to work:

```text
Mac
→ Public Internet
→ Instance Public IP
→ Internet Gateway
→ Route Table
→ Public Subnet
→ Security Rule TCP 22
→ Ubuntu SSH Service
→ Correct Private Key
```

If the error is:

```text
Operation timed out
```

think:

```text
Network problem
```

If the error is:

```text
Permission denied (publickey)
```

think:

```text
SSH key or username problem
```

---

# 24. Final Checklist

Before calling the server working, confirm:

```text
[ ] Instance is running
[ ] Instance has public IPv4
[ ] Subnet is public
[ ] Route table has 0.0.0.0/0 → Internet Gateway
[ ] Internet Gateway is enabled
[ ] TCP 22 is allowed
[ ] Private key is on Mac
[ ] Public key was pasted into OCI
[ ] SSH login works
[ ] cloud-init completed
[ ] apt update works
[ ] Docker works
[ ] Nginx works
```

Final login command:

```bash
ssh -i ~/.ssh/oci_ubuntu_server_lab ubuntu@<PUBLIC_IP>
```
