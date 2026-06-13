# AWS Hands-on Lab (Production Grade)

# Assign a Custom Private IP (10.10.10.10) and Attach Multiple Private IPs to an EC2 Instance

---

# Lab Objective

In this lab, you will build a complete private networking environment from scratch and perform the following tasks:

* Create a custom VPC
* Create a private subnet only
* Create a private route table
* Create a security group
* Launch an EC2 instance inside the private subnet
* Assign a custom private IP (`10.10.10.10`) during launch
* Attach a second private IP (`10.10.10.20`)
* Verify both IPs using the AWS Console and Linux commands
* Understand what happens after reboot, stop/start, and termination

This lab follows enterprise best practices and does not skip any steps.

---

# Lab Topology

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/9fa4edc9-db11-4cef-8c13-248b5f1b39a3" />


---

# Prerequisites

* AWS Account
* Administrator IAM User (for lab)
* Existing EC2 Key Pair
* AWS Region: ap-south-1 (recommended)
* Session Manager or Bastion Host (for connecting to private EC2)

---

# Understanding the CIDR

We need the EC2 to have:

```text
10.10.10.10
```

Therefore our subnet must contain this IP.

We choose:

```text
10.10.10.0/24
```

Range:

```text
Network Address

10.10.10.0

First Host

10.10.10.1

Last Host

10.10.10.254

Broadcast

10.10.10.255
```

Therefore

```text
10.10.10.10

✅ Valid
```

---

# PART 1 - Create VPC

## Step 1

Open AWS Console

```
Search

↓

VPC

↓

Your VPCs

↓

Create VPC
```

---

## Step 2

Select

```
VPC Only
```

---

## Step 3

Enter

```
Name

Private-VPC
```

---

## Step 4

IPv4 CIDR

```
10.10.0.0/16
```

---

## Step 5

IPv6

```
No IPv6 CIDR
```

---

## Step 6

Tenancy

```
Default
```

---

## Step 7

Click

```
Create VPC
```

---

## Verify

```
VPC ID

vpc-xxxxxxxx

CIDR

10.10.0.0/16
```

---

# PART 2 - Create Private Subnet

Open

```
VPC

↓

Subnets

↓

Create Subnet
```

---

Select VPC

```
Private-VPC
```

---

Subnet Name

```
Private-Subnet
```

---

Availability Zone

```
ap-south-1a
```

---

CIDR

```
10.10.10.0/24
```

---

Click

```
Create Subnet
```

---

Verify

```
Subnet CIDR

10.10.10.0/24
```

---

# PART 3 - Disable Public IP Assignment

Open

```
Private-Subnet

↓

Actions

↓

Edit Subnet Settings
```

Verify

```
Enable Auto Assign Public IPv4

❌ Disabled
```

Click

```
Save
```

This ensures all instances launched into this subnet remain private.

---

# PART 4 - Create Private Route Table

Open

```
VPC

↓

Route Tables

↓

Create Route Table
```

---

Name

```
Private-RT
```

---

Select VPC

```
Private-VPC
```

---

Click

```
Create Route Table
```

---

# PART 5 - Associate Route Table

Open

```
Private-RT

↓

Subnet Associations

↓

Edit
```

Select

```
Private-Subnet
```

Click

```
Save
```

---

# PART 6 - Verify Routes

Open

```
Private-RT

↓

Routes
```

Expected

```
Destination

10.10.0.0/16

Target

local
```

There should NOT be

```
0.0.0.0/0
```

because this is a private subnet.

---

# PART 7 - Create Security Group

Open

```
EC2

↓

Security Groups

↓

Create Security Group
```

---

Name

```
Private-SG
```

---

Description

```
Private EC2 Security Group
```

---

VPC

```
Private-VPC
```

---

Inbound Rule

```
SSH

Port

22

Source

10.10.0.0/16
```

(Optional)

```
ICMP

All

10.10.0.0/16
```

---

Click

```
Create Security Group
```

---

# PART 8 - Launch EC2

Open

```
EC2

↓

Launch Instance
```

---

Name

```
Private-Server
```

---

AMI

```
Amazon Linux 2023
```

---

Instance Type

```
t3.micro
```

---

Key Pair

```
Select Existing Key Pair or create new
```

---

Network

```
Private-VPC
```

---

Subnet

```
Private-Subnet
```

---

Auto Assign Public IP

```
Disable
```

---

Security Group

```
Private-SG
```

---

# PART 9 - Assign Custom Private IP

Expand

```
Advanced Network Configuration
```

Choose

```
Private IPv4 Address

↓

Enter Custom IP
```

Enter

```
10.10.10.10
```

Click

```
Launch Instance
```

---

# PART 10 - Verify Custom Private IP

Open

```
EC2

↓

Private-Server

↓

Networking
```

Verify

```
Primary Private IPv4

10.10.10.10
```

✅ Success

---

# PART 11 - Attach Second Private IP if You Want 

Open

```
EC2

↓

Private-Server

↓

Networking Tab

↓

Network Interface ID
```

Click the ENI.

---

Open

```
Actions

↓

Manage IP Addresses
```

---

Click

```
Assign New IP Address
```

---

Choose

```
Assign Specific IP
```

Enter

```
10.10.10.20
```

Click

```
Save
```

---

# PART 12 - Verify from AWS Console

Open

```
Network Interface

↓

Private IPv4 Addresses
```

Expected

```
Primary

10.10.10.10
```

```
Secondary

10.10.10.20
```

---



# PART 13 - Secure Access to Private EC2 Using Pritunl VPN and this is if You want Electic IP otherwise You Can Continue 

Since the EC2 instance is deployed in a **private subnet**, it cannot be accessed directly from the internet.

In production environments, organizations typically use one of the following methods:

* AWS Client VPN
* Pritunl VPN
* OpenVPN
* Bastion Host
* AWS Systems Manager Session Manager
* AWS Direct Connect

In this lab, we will deploy a **Pritunl VPN Server** in a **public subnet** and securely access the private EC2 instance through an encrypted VPN tunnel.

---


# Step 13.1 - We Already Launch Pritunl VPN EC2

Launch a new EC2 instance.

```
Name

Pritunl-VPN
```

AMI

```
Ubuntu 24.04 LTS
```

Instance Type

```
t3.micro
```

Network

```
Private-VPC
```

Subnet

```
Public-Subnet
```

Auto Assign Public IP

```
Enable
```

Security Group

```
Pritunl-SG
```

Launch the instance.

---

# Step 13.2 - Configure Security Group

Inbound Rules

| Type       | Port | Source         |
| ---------- | ---- | -------------- |
| SSH        | 22   | Your Public IP |
| HTTP       | 80   | 0.0.0.0/0      |
| HTTPS      | 443  | 0.0.0.0/0      |
| Custom UDP | 1194 | 0.0.0.0/0      |

Outbound

```
Allow All
```

---

# Step 13.3 - Install MongoDB

```bash
sudo apt update

sudo apt install gnupg curl -y

curl -fsSL https://pgp.mongodb.com/server-8.0.asc | \
sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg \
--dearmor

echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

sudo apt update

sudo apt install mongodb-org -y

sudo systemctl enable mongod

sudo systemctl start mongod
```

Verify

```bash
sudo systemctl status mongod
```

---

# Step 13.4 - Install Pritunl

```bash
curl -fsSL https://raw.githubusercontent.com/pritunl/pgp/master/pritunl_repo_pub.asc | \
gpg --dearmor | sudo tee /usr/share/keyrings/pritunl.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/pritunl.gpg] https://repo.pritunl.com/stable/apt noble main" | \
sudo tee /etc/apt/sources.list.d/pritunl.list

sudo apt update

sudo apt install pritunl -y

sudo systemctl enable pritunl

sudo systemctl start pritunl
```

---

# Step 13.5 - Configure MongoDB URI

Edit

```bash
sudo nano /etc/pritunl.conf
```

Update

```json
{
    "mongodb_uri":"mongodb://localhost:27017/pritunl"
}
```

Restart

```bash
sudo systemctl restart pritunl
```

Retrieve setup key

```bash
sudo pritunl setup-key
```

Retrieve default credentials

```bash
sudo pritunl default-password
```

Open

```
https://Public-IP
```

<img width="3340" height="1824" alt="image" src="https://github.com/user-attachments/assets/9e40ce21-fbab-4a4a-ab42-eda20fbfe5d7" />

Complete the initial setup.

---

# Step 13.6 - Create VPN Server

| Setting           | Value                   |
| ----------------- | ----------------------- |
| Name              | AWS-VPN                 |
| Protocol          | UDP                     |
| Port              | 1194                    |
| Virtual Network   | 172.31.250.0/24         |
| DNS               | 8.8.8.8                 |
| WireGuard         | Disabled                |
| IPv6              | Disabled                |
| Route All Traffic | Disabled (Split Tunnel) |

Start the server.

---

# Step 13.7 - Add Route

```
Servers

↓

AWS-VPN

↓

Add Route
```

| Field       | Value        |
| ----------- | ------------ |
| Network     | 10.10.0.0/16 |
| NAT         | Enabled      |
| Net Gateway | Blank        |

Save.

---

# Step 13.8 - Create Organization and User

Create Organization

```
DevOps
```

Create User

```
username
```

Attach the organization to the VPN server.

Download the user profile.

---



# Step 13.9 - Configure Private EC2 Security Group

Add inbound rule

| Type | Port | Source          |
| ---- | ---- | --------------- |
| SSH  | 22   | 172.31.250.0/24 |

(Optional)

| Type | Source          |
| ---- | --------------- |
| ICMP | 172.31.250.0/24 |

---

# Step 13.10 - Connect from Laptop

Install the Pritunl Client. https://client.pritunl.com/#install

Import the downloaded profile.

Connect.

Your laptop receives a VPN IP such as

```
172.31.250.2
```

Verify

```bash
ping 10.10.10.10
```

Connect securely

```bash
ssh -i <Key Name> ubuntu@10.10.10.10
```

---

# Result

* The private EC2 remains inaccessible from the public internet.
* Only authenticated VPN users can access the VPC.
* SSH keys remain on the client machine and are never copied to the VPN server.
* Split Tunnel ensures only traffic destined for `10.10.0.0/16` traverses the VPN, while normal internet browsing continues through the local ISP.
* This architecture closely matches enterprise production environments for secure administrative access to private workloads.


# PART 14 - Verify Inside Linux

Check all IP addresses

```bash
ip addr
```

Expected

```
inet 10.10.10.10

inet 10.10.10.20
```


---

Alternative

```bash
hostname -I
```

Output

```
10.10.10.10 10.10.10.20
```
<img width="939" height="95" alt="Screenshot 2026-06-13 at 6 06 12 PM" src="https://github.com/user-attachments/assets/20f43f67-5e59-449d-b396-36c6b97b4f6d" />



---

View routing

```bash
ip route
```

---

View interface

```bash
ip a show eth0
```

---



# PART 15 - Stop and Start Test

Stop the EC2 instance.

Start it again.

Check Networking.

Result:

```
Primary Private IP

10.10.10.10

✅ Same
```

```
Secondary Private IP

10.10.10.20

✅ Same
```

Private IPs remain attached to the ENI.

---

# PART 16 - Reboot Test

Reboot the EC2.

Verify:

```
10.10.10.10

✅ Same
```

```
10.10.10.20

✅ Same
```

No changes occur.

---

# PART 17 - Termination Test

Terminate the EC2.

Result:

```
ENI Deleted

↓

10.10.10.10 Released

↓

10.10.10.20 Released
```

The IPs return to the subnet's available pool.

---

# Common Errors

## Invalid IP

Reason:

```
IP outside subnet range
```

Example

```
Subnet

10.10.10.0/24

Trying

192.168.1.10

❌ Invalid
```

---

## IP Already Assigned

```
Address already in use
```

Choose another available IP.

---

## Cannot Connect

Reason:

Private subnet has no Internet access.

Use:

* Session Manager
* Bastion Host
* VPN

---

# Interview Questions

### Can you assign a custom private IP to EC2?

Yes. During instance launch or later through the Elastic Network Interface (ENI), provided the IP belongs to the subnet and is not already in use.

---

### Can one EC2 have multiple private IPs?

Yes. AWS supports multiple secondary private IP addresses on a single ENI, subject to instance limits.

---

### Will the private IP change after Stop and Start?

**No.** Both primary and secondary private IP addresses remain attached to the ENI and are retained across stop/start operations.

---

### Will the public IP change after Stop and Start?

If it is an **auto-assigned public IP**, **Yes**, it changes.

If it is an **Elastic IP**, **No**, it remains the same.

---

### Why use multiple private IPs?

* High Availability
* Virtual IP failover
* Multiple applications
* Network appliances
* Database clusters
* Legacy application migration

---

# Final Learning Outcome

After completing this lab, you will be able to:

* Build a custom VPC from scratch
* Design a private-only subnet
* Launch a private EC2 instance
* Assign a custom private IP (`10.10.10.10`)
* Attach additional private IPs (`10.10.10.20`, etc.)
* Verify networking from the AWS Console, CLI, and Linux
* Understand ENI behavior during reboot, stop/start, and termination
* Confidently answer AWS networking interview questions related to custom private IP addressing and multi-IP EC2 instances.
