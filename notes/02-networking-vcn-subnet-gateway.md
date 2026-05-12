# 02 OCI Networking Setup: VCN, Subnet, Internet Gateway, Route Table, and Security Rules

## Goal

Set up the networking foundation for a public Ubuntu server on Oracle Cloud Infrastructure.

This note covers:

```text
VCN creation
Public subnet creation
Internet Gateway setup
Route table setup
Security List or NSG setup
SSH port validation
Network troubleshooting logic
```

Project setup belongs in:

```text
notes/01-project-setup-ssh-cloud-init.md
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

# 1. Network Architecture

For a simple Ubuntu server lab, use this structure:

| Resource | Example Name |
|---|---|
| VCN | `oci-ubuntu-lab-vcn` |
| Public Subnet | `public-subnet-01` |
| Internet Gateway | `oci-ubuntu-lab-igw` |
| Route Table | `public-subnet-route-table` |
| Security List or NSG | `public-ssh-web-rules` |
| Compute Instance | `oci-ubuntu-server-lab-01` |

Recommended CIDR blocks:

| Network Item | CIDR |
|---|---|
| VCN | `10.0.0.0/16` |
| Public Subnet | `10.0.0.0/24` |
| Optional Private Subnet | `10.0.1.0/24` |

Explanation:

```text
10.0.0.0/16 is the larger private network.
10.0.0.0/24 is a smaller public subnet inside that VCN.
10.0.1.0/24 can be used later for private resources such as databases.
```

The intended traffic path is:

```text
Mac Terminal
   ↓ SSH TCP 22
Instance Public IP
   ↓
Internet Gateway
   ↓
Route Table
   ↓
Public Subnet
   ↓
Ubuntu Instance VNIC
```

---

# 2. Create the VCN

Go to:

```text
Oracle Cloud Console → Networking → Virtual Cloud Networks → Create VCN
```

Use:

```text
Name: oci-ubuntu-lab-vcn
Compartment: Your selected compartment
IPv4 CIDR Block: 10.0.0.0/16
DNS Resolution: Enabled
DNS Label: ociubuntulabvcn
```

Then click:

```text
Create VCN
```

## Why this matters

The VCN is the main private cloud network.

Subnets, gateways, route tables, security rules, and instances live inside the VCN.

A VCN by itself does not automatically give internet access.

Internet access requires:

```text
Internet Gateway
Route Table Rule
Public Subnet
Public IP
Security Rule
```

---

# 3. Create the Public Subnet

Inside the VCN, go to:

```text
VCN → Subnets → Create Subnet
```

Use:

```text
Name: public-subnet-01
Create in Compartment: Your selected compartment
Subnet Type: Regional
IPv4 CIDR Block: 10.0.0.0/24
Subnet Access: Public Subnet
DNS Resolution: Use DNS hostnames in this subnet
DNS Label: publicsubnet01
DHCP Options: Default DHCP Options
Route Table: Default route table for now, or custom public route table
Security List: Default Security List for now, or custom security list
```

Then click:

```text
Create Subnet
```

Important:

```text
Choose Public Subnet, not Private Subnet.
```

A public subnet allows instances in the subnet to receive public IPv4 addresses.

For direct SSH from your Mac, the instance must be in a public subnet and must have a public IP address.

---

# 4. Create or Verify the Internet Gateway

Go to:

```text
VCN → Internet Gateways
```

Create an Internet Gateway if one does not already exist.

Use:

```text
Name: oci-ubuntu-lab-igw
Compartment: Your selected compartment
Enabled: Yes
```

Then click:

```text
Create Internet Gateway
```

## Why this matters

The Internet Gateway allows traffic between the VCN and the public internet.

The gateway alone is not enough.

The subnet route table must also contain a route rule that sends internet-bound traffic to the Internet Gateway.

---

# 5. Create a Public Route Table

This is the critical step.

Go to:

```text
VCN → Route Tables → Create Route Table
```

Use:

```text
Name: public-subnet-route-table
Compartment: Your selected compartment
```

Add this route rule:

```text
Destination CIDR Block: 0.0.0.0/0
Target Type: Internet Gateway
Target: oci-ubuntu-lab-igw
Description: Public subnet internet route
```

Then create the route table.

## Meaning of the route rule

```text
0.0.0.0/0 → Internet Gateway
```

means:

```text
Send all internet-bound traffic from this subnet to the Internet Gateway.
```

Without this rule, an instance can have a public IP address but still be unreachable from the internet.

---

# 6. Attach the Route Table to the Public Subnet

Go to:

```text
VCN → Subnets → public-subnet-01
```

Click:

```text
Edit
```

Change the route table to:

```text
public-subnet-route-table
```

Then save.

After this, the subnet should show:

```text
Subnet: public-subnet-01
Subnet Access: Public Subnet
Route Table: public-subnet-route-table
```

The route table should contain:

```text
Destination: 0.0.0.0/0
Target Type: Internet Gateway
Target: oci-ubuntu-lab-igw
```

---

# 7. Route Table Troubleshooting

## Problem: No route rules

If the route table shows:

```text
No items to display
```

then there are no route rules.

That means the public subnet has no internet route.

SSH will likely fail with:

```text
Operation timed out
```

Fix:

```text
Destination CIDR Block: 0.0.0.0/0
Target Type: Internet Gateway
Target: oci-ubuntu-lab-igw
```

---

## Problem: Private IP target route error

OCI may show this error:

```text
Rules in the route table must use private IP as a target.
Or the route table can be empty.
```

Meaning:

```text
OCI is treating that route table as a special routing table for internal routing.
```

For a normal public Ubuntu server, do not use a private IP target.

Fix:

```text
Create a new clean route table.
Add 0.0.0.0/0 → Internet Gateway.
Attach the new route table to public-subnet-01.
```

Correct public server route:

```text
Destination: 0.0.0.0/0
Target Type: Internet Gateway
Target: VCN Internet Gateway
```

---

# 8. Configure Security Rules

OCI networking has two main security-rule options:

```text
Security Lists
Network Security Groups
```

Security Lists apply to the subnet.

Network Security Groups apply to specific VNICs or resources.

For a simple lab, either can work.

The important thing is that TCP port `22` must be allowed for SSH.

---

# 9. Required SSH Ingress Rule

Recommended safer version:

```text
Source CIDR: YOUR_PUBLIC_IP/32
Protocol: TCP
Destination Port Range: 22
Stateless: No
Description: Allow SSH from my IP
```

Temporary setup version:

```text
Source CIDR: 0.0.0.0/0
Protocol: TCP
Destination Port Range: 22
Stateless: No
Description: Temporary SSH access
```

Important:

```text
0.0.0.0/0 for SSH means anyone on the internet can try to connect to port 22.
```

This may be acceptable temporarily while setting up the server.

After setup, restrict SSH to:

```text
YOUR_PUBLIC_IP/32
```

Example format:

```text
203.0.113.10/32
```

---

# 10. Optional Web Server Ingress Rules

If the server will host a website, Nginx, Caddy, Docker app, or API, add these later.

HTTP:

```text
Source CIDR: 0.0.0.0/0
Protocol: TCP
Destination Port Range: 80
Stateless: No
Description: Allow HTTP
```

HTTPS:

```text
Source CIDR: 0.0.0.0/0
Protocol: TCP
Destination Port Range: 443
Stateless: No
Description: Allow HTTPS
```

Do not open random Docker ports publicly unless needed.

Avoid exposing ports such as:

```text
5432  # PostgreSQL
6379  # Redis
8000  # Development servers
8080  # Development servers
9000  # Admin tools
```

Better pattern:

```text
Internet → 80/443 → Nginx or Caddy → Docker app internally
```

---

# 11. Required Egress Rule

The server needs outbound internet access for package updates, Docker pulls, Git, and downloads.

Use an egress rule like:

```text
Destination CIDR: 0.0.0.0/0
Protocol: All
Stateless: No
Description: Allow outbound internet access
```

This allows commands such as:

```bash
sudo apt update
sudo apt upgrade
curl https://example.com
git clone <repository-url>
docker pull ubuntu
```

---

# 12. Check Instance Networking Settings After Compute Creation

After the compute instance is created, confirm these details:

```text
Instance State: Running
Subnet: public-subnet-01
Public IPv4 Address: Present
Private IPv4 Address: Present
Route Table: public-subnet-route-table or correct public route table
Network Security Group: Has SSH rule if attached
```

Important:

```text
A private IP is not enough for direct SSH from your Mac.
The instance needs a public IPv4 address.
```

---

# 13. Test Port 22 from Mac

Before full SSH login, test whether port `22` is reachable.

From Mac Terminal:

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

If the test times out, check:

```text
1. Instance is running
2. Instance has a public IPv4 address
3. Subnet is public
4. Internet Gateway exists and is enabled
5. Route table has 0.0.0.0/0 → Internet Gateway
6. Security List allows TCP 22
7. Network Security Group allows TCP 22 if attached
8. Local network is not blocking outbound SSH
```

---

# 14. SSH Traffic Path Checklist

SSH works only if this full path is complete:

```text
Local Mac
   ↓
Public Internet
   ↓
Instance Public IP
   ↓
VCN Internet Gateway
   ↓
Route Table Rule: 0.0.0.0/0 → Internet Gateway
   ↓
Public Subnet
   ↓
Security Rule: TCP 22 allowed
   ↓
Ubuntu Instance
```

If any link in that chain is missing, SSH may fail.

---

# 15. Network Validation Checklist

Before SSH, verify:

```text
[ ] VCN exists
[ ] Public subnet exists
[ ] Subnet access is Public Subnet
[ ] Internet Gateway exists
[ ] Internet Gateway is enabled
[ ] Route table has 0.0.0.0/0 → Internet Gateway
[ ] Public subnet is attached to the correct route table
[ ] Security List or NSG allows TCP 22
[ ] Instance has a public IPv4 address
[ ] Instance state is Running
```

Then test:

```bash
nc -vz <PUBLIC_IP> 22
```

Then SSH:

```bash
ssh -i ~/.ssh/oci_ubuntu_server_lab ubuntu@<PUBLIC_IP>
```

---

# 16. Final Working Pattern

The final networking pattern is:

```text
Public IP
+ Public Subnet
+ Internet Gateway
+ Route Table: 0.0.0.0/0 → Internet Gateway
+ Security Rule: TCP 22
= SSH access from Mac
```

For web server work later:

```text
Public IP
+ Public Subnet
+ Internet Gateway
+ Route Table
+ Security Rules: TCP 80 and TCP 443
+ Nginx or Caddy
= Public website or API server
```

---

# 17. Best Practice Notes

## SSH

During initial setup:

```text
SSH from 0.0.0.0/0 may be used temporarily.
```

After confirming access:

```text
Restrict SSH to YOUR_PUBLIC_IP/32.
```

## Databases

Do not expose databases directly to the internet.

Avoid public access to:

```text
PostgreSQL: 5432
MySQL: 3306
Redis: 6379
MongoDB: 27017
```

Use:

```text
Private subnet
SSH tunnel
VPN
Bastion host
```

## Route Tables

A public subnet must have:

```text
0.0.0.0/0 → Internet Gateway
```

A private subnet usually uses:

```text
0.0.0.0/0 → NAT Gateway
```

for outbound-only internet access.

---

# 18. Summary

To make an OCI Ubuntu instance reachable from a Mac through SSH, the network must have:

```text
1. VCN
2. Public subnet
3. Internet Gateway
4. Route table rule to the Internet Gateway
5. Security rule allowing TCP 22
6. Instance public IPv4 address
```

The biggest lesson:

```text
A public IP address alone does not guarantee internet reachability.
The route table must explicitly send internet traffic to the Internet Gateway.
```
