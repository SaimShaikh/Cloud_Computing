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

```text
                           AWS Cloud

                     Custom VPC
                     10.10.0.0/16
                            │
                            │
                 ┌──────────┴──────────┐
                 │                     │
                 │ Private Route Table │
                 │                     │
                 └──────────┬──────────┘
                            │
                            │
                 Private Subnet
                  10.10.10.0/24
                            │
                            │
                  EC2 Instance (Private)
                            │
             ┌──────────────┴──────────────┐
             │                             │
             │                             │
     Primary Private IP            Secondary Private IP
        10.10.10.10                   10.10.10.20
```

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
Select Existing Key Pair
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

# PART 11 - Attach Second Private IP

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

# PART 13 - Connect to EC2

Since this EC2 has no public IP, connect using:

* AWS Systems Manager Session Manager
* Bastion Host
* VPN
* Direct Connect

Recommended:

```
EC2

↓

Connect

↓

Session Manager

↓

Connect
```

---

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

# PART 15 - Verify Using AWS CLI

Describe instance

```bash
aws ec2 describe-instances \
--instance-ids i-xxxxxxxx
```

---

Describe ENI

```bash
aws ec2 describe-network-interfaces \
--network-interface-ids eni-xxxxxxxx
```

Expected

```json
"PrivateIpAddresses":[
 {
   "PrivateIpAddress":"10.10.10.10",
   "Primary":true
 },
 {
   "PrivateIpAddress":"10.10.10.20",
   "Primary":false
 }
]
```

---

# PART 16 - Stop and Start Test

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

# PART 17 - Reboot Test

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

# PART 18 - Termination Test

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
