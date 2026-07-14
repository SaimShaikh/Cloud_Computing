# AWS Transit Gateway (TGW) Hub and Spoke Architecture Lab

## Overview

This lab demonstrates how to connect multiple Amazon VPCs using an AWS Transit Gateway (TGW) in a Hub and Spoke architecture.

By the end of this lab, you will have:

- 4 VPCs
- 1 Subnet in each VPC
- 1 Internet Gateway attached to each VPC
- 1 Transit Gateway
- 4 Transit Gateway Attachments
- Route tables configured for inter-VPC communication
- One EC2 instance deployed in each VPC
- End-to-end connectivity using private IP addresses

---

## Architecture

```
                         Internet
                             |
     -----------------------------------------------------
     |            |             |                       |
    IGW          IGW           IGW                    IGW
     |            |             |                       |
+------------+ +------------+ +------------+ +------------+
|   VPC-A    | |   VPC-B    | |   VPC-C    | |   VPC-D    |
|10.1.0.0/16 | |10.2.0.0/16 | |10.3.0.0/16 | |10.4.0.0/16 |
|            | |            | |            | |            |
|10.1.1.0/24 | |10.2.1.0/24 | |10.3.1.0/24 | |10.4.1.0/24 |
|    EC2     | |    EC2     | |    EC2     | |    EC2     |
+------------+ +------------+ +------------+ +------------+
      \              |               |               /
       \             |               |              /
        \            |               |             /
         -----------------------------------------
                    AWS Transit Gateway
```

---

## Prerequisites

Before starting this lab, make sure you have:

- AWS Account
- IAM User with Administrator Access
- AWS Console Access
- Key Pair (.pem)
- Region: **ap-south-1 (Mumbai)**

---

## Network Planning

### VPC Planning

| VPC | CIDR |
|-----|------|
| VPC-A | 10.1.0.0/16 |
| VPC-B | 10.2.0.0/16 |
| VPC-C | 10.3.0.0/16 |
| VPC-D | 10.4.0.0/16 |

### Subnet Planning

| VPC | Subnet | CIDR |
|-----|--------|------|
| VPC-A | VPC-A-Subnet | 10.1.1.0/24 |
| VPC-B | VPC-B-Subnet | 10.2.1.0/24 |
| VPC-C | VPC-C-Subnet | 10.3.1.0/24 |
| VPC-D | VPC-D-Subnet | 10.4.1.0/24 |

---

## Step 1 — Create VPC-A

Navigate to:

```
VPC Console → Your VPCs → Create VPC
```

Choose:

- Resources to create: **VPC Only**

Fill the details:

| Parameter | Value |
|-----------|-------|
| Name | VPC-A |
| IPv4 CIDR | 10.1.0.0/16 |
| IPv6 | None |
| Tenancy | Default |

Click **Create VPC**.

---

## Step 2 — Create VPC-B

Repeat the same steps.

| Parameter | Value |
|-----------|-------|
| Name | VPC-B |
| CIDR | 10.2.0.0/16 |

Create VPC.

---

## Step 3 — Create VPC-C

| Parameter | Value |
|-----------|-------|
| Name | VPC-C |
| CIDR | 10.3.0.0/16 |

Create VPC.

---

## Step 4 — Create VPC-D

| Parameter | Value |
|-----------|-------|
| Name | VPC-D |
| CIDR | 10.4.0.0/16 |

Create VPC.

**Expected Result:**

```
VPC-A
VPC-B
VPC-C
VPC-D
```

---

## Step 5 — Create Subnets

Navigate to:

```
VPC Console → Subnets → Create Subnet
```

### Subnet for VPC-A

| Parameter | Value |
|-----------|-------|
| VPC | VPC-A |
| Name | VPC-A-Subnet |
| AZ | ap-south-1a |
| CIDR | 10.1.1.0/24 |

Click **Create Subnet**.

### Subnet for VPC-B

| Parameter | Value |
|-----------|-------|
| VPC | VPC-B |
| Name | VPC-B-Subnet |
| AZ | ap-south-1a |
| CIDR | 10.2.1.0/24 |

### Subnet for VPC-C

| Parameter | Value |
|-----------|-------|
| VPC | VPC-C |
| Name | VPC-C-Subnet |
| AZ | ap-south-1a |
| CIDR | 10.3.1.0/24 |

### Subnet for VPC-D

| Parameter | Value |
|-----------|-------|
| VPC | VPC-D |
| Name | VPC-D-Subnet |
| AZ | ap-south-1a |
| CIDR | 10.4.1.0/24 |

**Expected Result:**

```
VPC-A
└──10.1.1.0/24

VPC-B
└──10.2.1.0/24

VPC-C
└──10.3.1.0/24

VPC-D
└──10.4.1.0/24
```

---

## Step 6 — Create Internet Gateway

Navigate to:

```
VPC Console → Internet Gateways → Create Internet Gateway
```

Create the following:

| Name |
|------|
| IGW-A |
| IGW-B |
| IGW-C |
| IGW-D |

After creating each Internet Gateway, click **Actions → Attach to VPC**.

Attach them as follows:

| Internet Gateway | Attach To |
|-------------------|-----------|
| IGW-A | VPC-A |
| IGW-B | VPC-B |
| IGW-C | VPC-C |
| IGW-D | VPC-D |

**Expected Result:**

```
IGW-A → VPC-A
IGW-B → VPC-B
IGW-C → VPC-C
IGW-D → VPC-D
```

---

## Step 7 — Create Route Tables

When a VPC is created, AWS automatically creates a **Main Route Table** for that VPC.

For this lab, we will use the existing route tables instead of creating new ones.

Navigate to:

```
VPC Console → Route Tables
```

Rename the automatically created route tables for easier identification.

| Route Table | VPC |
|-------------|-----|
| VPC-A | VPC-A |
| VPC-B | VPC-B |
| VPC-C | VPC-C |
| VPC-D | VPC-D |

---

## Step 8 — Add Internet Gateway Route

Each VPC should have Internet access.

Navigate to:

```
Route Table → Routes → Edit Routes → Add Route
```

Add the following route:

| Destination | Target |
|-------------|--------|
| 0.0.0.0/0 | Internet Gateway |

Repeat for all four route tables.

**Expected Route Tables:**

**VPC-A**

| Destination | Target |
|-------------|--------|
| 10.1.0.0/16 | local |
| 0.0.0.0/0 | IGW-A |

**VPC-B**

| Destination | Target |
|-------------|--------|
| 10.2.0.0/16 | local |
| 0.0.0.0/0 | IGW-B |

**VPC-C**

| Destination | Target |
|-------------|--------|
| 10.3.0.0/16 | local |
| 0.0.0.0/0 | IGW-C |

**VPC-D**

| Destination | Target |
|-------------|--------|
| 10.4.0.0/16 | local |
| 0.0.0.0/0 | IGW-D |

---

## Step 9 — Create Transit Gateway

Navigate to:

```
VPC Console → Transit Gateways → Create Transit Gateway
```

Configure the following:

| Parameter | Value |
|-----------|-------|
| Name | TGW-Hub-Demo |
| Description | Hub and Spoke Architecture |
| Amazon ASN | 64512 (Default) |
| Auto Accept Shared Attachments | Disable |
| Default Route Table Association | Enable |
| Default Route Table Propagation | Enable |
| DNS Support | Enable |
| VPN ECMP Support | Enable |

Click **Create Transit Gateway**.

Wait until the Transit Gateway state changes to `Available`.

---

## Step 10 — Create Transit Gateway Attachments

Navigate to:

```
Transit Gateway → Attachments → Create Transit Gateway Attachment
```

Create one attachment for each VPC.

**Attachment 1**

| Parameter | Value |
|-----------|-------|
| Name | TGW-Attachment-VPC-A |
| Resource Type | VPC |
| Transit Gateway | TGW-Hub-Demo |
| VPC | VPC-A |
| Subnet | VPC-A-Subnet |

Click **Create Attachment**.

**Attachment 2**

| Parameter | Value |
|-----------|-------|
| Name | TGW-Attachment-VPC-B |
| Resource Type | VPC |
| Transit Gateway | TGW-Hub-Demo |
| VPC | VPC-B |
| Subnet | VPC-B-Subnet |

**Attachment 3**

| Parameter | Value |
|-----------|-------|
| Name | TGW-Attachment-VPC-C |
| Resource Type | VPC |
| Transit Gateway | TGW-Hub-Demo |
| VPC | VPC-C |
| Subnet | VPC-C-Subnet |

**Attachment 4**

| Parameter | Value |
|-----------|-------|
| Name | TGW-Attachment-VPC-D |
| Resource Type | VPC |
| Transit Gateway | TGW-Hub-Demo |
| VPC | VPC-D |
| Subnet | VPC-D-Subnet |

Wait until all attachments show `Available`.

---

## Step 11 — Add Other VPCs' CIDR Blocks to Each Route Table

No subnet association changes are needed here — each VPC keeps its own subnet tied to its own route table as created by default.

The only thing required in this step is: for every VPC's route table, add a route for **each of the other three VPCs' CIDR blocks**, pointing to the Transit Gateway.

Example — in **VPC-A's** route table, add the CIDR ranges of **VPC-B, VPC-C, and VPC-D**, each pointing to the Transit Gateway. Repeat this pattern for VPC-B, VPC-C, and VPC-D (each pointing to the other three).

Navigate to:

```
VPC Console → Route Tables → Routes → Edit Routes
```

**VPC-A Route Table**

Keep the existing routes and add:

| Destination | Target |
|-------------|--------|
| 10.2.0.0/16 | Transit Gateway |
| 10.3.0.0/16 | Transit Gateway |
| 10.4.0.0/16 | Transit Gateway |

Final Route Table:

| Destination | Target |
|-------------|--------|
| 10.1.0.0/16 | local |
| 10.2.0.0/16 | Transit Gateway |
| 10.3.0.0/16 | Transit Gateway |
| 10.4.0.0/16 | Transit Gateway |
| 0.0.0.0/0 | Internet Gateway |

**VPC-B Route Table**

| Destination | Target |
|-------------|--------|
| 10.2.0.0/16 | local |
| 10.1.0.0/16 | Transit Gateway |
| 10.3.0.0/16 | Transit Gateway |
| 10.4.0.0/16 | Transit Gateway |
| 0.0.0.0/0 | Internet Gateway |

**VPC-C Route Table**

| Destination | Target |
|-------------|--------|
| 10.3.0.0/16 | local |
| 10.1.0.0/16 | Transit Gateway |
| 10.2.0.0/16 | Transit Gateway |
| 10.4.0.0/16 | Transit Gateway |
| 0.0.0.0/0 | Internet Gateway |

**VPC-D Route Table**

| Destination | Target |
|-------------|--------|
| 10.4.0.0/16 | local |
| 10.1.0.0/16 | Transit Gateway |
| 10.2.0.0/16 | Transit Gateway |
| 10.3.0.0/16 | Transit Gateway |
| 0.0.0.0/0 | Internet Gateway |

**Expected Result:**

- 4 VPCs created ✅
- 4 Subnets created ✅
- 4 Internet Gateways attached ✅
- 4 Route Tables configured ✅
- 4 TGW Attachments available ✅
- All Route Tables contain the other three VPCs' CIDR blocks pointing to TGW ✅

---

## Step 12 — Launch EC2 Instances

Launch one EC2 instance in each VPC.

Navigate to:

```
EC2 Console → Instances → Launch Instance
```

**EC2 Configuration** (use for all four instances):

| Parameter | Value |
|-----------|-------|
| Name | EC2-VPC-A (change accordingly) |
| AMI | Ubuntu Server 24.04 LTS |
| Instance Type | t2.micro |
| Key Pair | Select your existing Key Pair |
| VPC | Select respective VPC |
| Subnet | Select respective subnet |
| Auto Assign Public IP | Enable |
| Security Group | Create New |

Repeat the same process for all VPCs.

**EC2 Details**

| Instance Name | VPC | Subnet |
|---------------|-----|--------|
| EC2-VPC-A | VPC-A | VPC-A-Subnet |
| EC2-VPC-B | VPC-B | VPC-B-Subnet |
| EC2-VPC-C | VPC-C | VPC-C-Subnet |
| EC2-VPC-D | VPC-D | VPC-D-Subnet |

Wait until every instance reaches `Running` and `Status Check: 3/3 Checks Passed`.

---

## Step 13 — Configure Security Groups

Each EC2 instance must allow SSH and ICMP traffic.

Navigate to:

```
EC2 → Security Groups → Inbound Rules → Edit Inbound Rules
```

**Rule 1**

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| SSH | TCP | 22 | My IP |

**Rule 2**

| Type | Protocol | Source |
|------|----------|--------|
| All ICMP - IPv4 | ICMP | 10.0.0.0/8 |

**(Optional) Allow SSH between EC2 instances**

| Type | Port | Source |
|------|------|--------|
| SSH | 22 | 10.0.0.0/8 |

**Expected Security Group:**

```
Inbound
- SSH        → My IP
- All ICMP   → 10.0.0.0/8
- SSH        → 10.0.0.0/8
```

---

## Step 14 — Verify EC2 Networking

Open each EC2 instance and verify:

| Parameter | Expected |
|-----------|----------|
| State | Running |
| Status Check | 3/3 Passed |
| Public IPv4 | Available |
| Private IPv4 | Available |

Example:

| EC2 | Private IP |
|-----|------------|
| EC2-VPC-A | 10.1.1.x |
| EC2-VPC-B | 10.2.1.x |
| EC2-VPC-C | 10.3.1.x |
| EC2-VPC-D | 10.4.1.x |

---

## Step 15 — Connect to EC2

Connect from your local machine using SSH:

```bash
ssh -i pvtkeys.pem ubuntu@<Public-IP>
```

Example:

```bash
ssh -i pvtkeys.pem ubuntu@13.xxx.xxx.xxx
```

Verify SSH login:

```
ubuntu@ip-10-x-x-x
```

Repeat for all EC2 instances.

---

## Step 16 — Verify Transit Gateway Connectivity

The Transit Gateway routes traffic using **Private IP Addresses**. Always use the **Private IP** while testing connectivity.

Example:

| Source | Destination |
|--------|-------------|
| EC2-A | 10.2.1.x |
| EC2-A | 10.3.1.x |
| EC2-A | 10.4.1.x |

**Ping Test** (from EC2-A):

```bash
ping 10.2.1.x
ping 10.3.1.x
ping 10.4.1.x
```

Repeat the same test from every EC2 instance.

**SSH Test** (from EC2-A):

```bash
ssh ubuntu@10.2.1.x
```

or

```bash
ssh ubuntu@10.3.1.x
```

If SSH authentication is required, copy your private key or use another authentication method.

---

## Important Note

For communication between VPCs connected through a Transit Gateway, always use the **Private IP Address**.

✅ Correct:

```bash
ping 10.3.1.191
```

❌ Incorrect:

```bash
ping 65.xxx.xxx.xxx
```

The Public IP uses the Internet Gateway and does **not** traverse the Transit Gateway.

---

## Step 17 — Final Verification Checklist

**VPCs**
- [ ] VPC-A Created
- [ ] VPC-B Created
- [ ] VPC-C Created
- [ ] VPC-D Created

**Subnets**
- [ ] VPC-A-Subnet
- [ ] VPC-B-Subnet
- [ ] VPC-C-Subnet
- [ ] VPC-D-Subnet

**Internet Gateways**
- [ ] IGW-A Attached
- [ ] IGW-B Attached
- [ ] IGW-C Attached
- [ ] IGW-D Attached

**Transit Gateway**
- [ ] Transit Gateway Available

**Transit Gateway Attachments**
- [ ] VPC-A Attachment Available
- [ ] VPC-B Attachment Available
- [ ] VPC-C Attachment Available
- [ ] VPC-D Attachment Available

**Route Tables** — each should contain:

**VPC-A**
```
10.1.0.0/16 → local
10.2.0.0/16 → TGW
10.3.0.0/16 → TGW
10.4.0.0/16 → TGW
0.0.0.0/0   → IGW
```

**VPC-B**
```
10.2.0.0/16 → local
10.1.0.0/16 → TGW
10.3.0.0/16 → TGW
10.4.0.0/16 → TGW
0.0.0.0/0   → IGW
```

**VPC-C**
```
10.3.0.0/16 → local
10.1.0.0/16 → TGW
10.2.0.0/16 → TGW
10.4.0.0/16 → TGW
0.0.0.0/0   → IGW
```

**VPC-D**
```
10.4.0.0/16 → local
10.1.0.0/16 → TGW
10.2.0.0/16 → TGW
10.3.0.0/16 → TGW
0.0.0.0/0   → IGW
```

---

## Step 18 — Cleanup

To avoid AWS charges, delete the resources in the following order:

1. Terminate all EC2 Instances
2. Delete Transit Gateway Attachments
3. Delete Transit Gateway
4. Detach Internet Gateways
5. Delete Internet Gateways
6. Delete Subnets
7. Delete Route Tables (only custom route tables, if any)
8. Delete VPCs

---

## Lab Completed Successfully

You have successfully created a Hub and Spoke Architecture using AWS Transit Gateway.

**Infrastructure Summary**

- 4 VPCs
- 4 Subnets
- 4 Internet Gateways
- 4 Route Tables
- 1 Transit Gateway
- 4 Transit Gateway Attachments
- 4 EC2 Instances
- End-to-End Private Connectivity using Transit Gateway
