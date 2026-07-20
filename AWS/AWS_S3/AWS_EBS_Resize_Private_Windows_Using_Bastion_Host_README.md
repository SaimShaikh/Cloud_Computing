# AWS Hands-on Lab: Increase Root EBS Volume (30 GB → 50 GB) on a Private Windows EC2 using a Bastion Host

## Objective

Create the following architecture:

- One Windows Bastion Host in a **Public Subnet**
- One Windows EC2 in a **Private Subnet**
- Private EC2 root EBS volume = **30 GB**
- Increase the EBS volume to **50 GB**
- Extend the Windows C: drive so the operating system reflects the new size.

---

# Architecture

<img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/09adc742-756f-48c4-a928-9bdb879d1a7a" />


---

# Prerequisites

- AWS Account
- One VPC
- One Public Subnet
- One Private Subnet
- Internet Gateway
- Route Tables
- Windows Bastion Host
- Windows Private EC2
- Security Groups configured
- RDP Client

---

# Step 1 - Create VPC

CIDR:
```
10.0.0.0/16
```

Create:

- Public Subnet
  - 10.0.1.0/24

- Private Subnet
  - 10.0.2.0/24

Attach an Internet Gateway to the VPC.

---

# Step 2 - Configure Route Tables

## Public Route Table

Destination | Target
------------|-------
10.0.0.0/16 | Local
0.0.0.0/0 | Internet Gateway

Associate with the Public Subnet.

## Private Route Table

Destination | Target
------------|-------
10.0.0.0/16 | Local

Associate with the Private Subnet.

---

# Step 3 - Launch Bastion Host

- Windows Server 2025
- Public Subnet
- Public IP Enabled
- t3.micro
- Security Group:
  - TCP 3389 from **your public IP only**

Wait until 2/2 status checks pass.

---

# Step 4 - Launch Private Windows EC2

- Windows Server 2025
- Private Subnet
- Public IP Disabled
- Root Volume: 30 GB gp3
- t3.micro

---

# Step 5 - Configure Security Groups

## Bastion SG

Inbound:
- TCP 3389 -> Your Public IP

Outbound:
- All Traffic

## Private EC2 SG

Inbound:
- TCP 3389 -> Bastion SG

Outbound:
- All Traffic

---

# Step 6 - Connect

1. RDP from your laptop to the Bastion Host.
2. From the Bastion Host, launch Remote Desktop Connection.
3. Connect to the **private IP** of the private Windows EC2.
4. Login using the Administrator credentials.

---

# Step 7 - Verify Initial Disk

```powershell
Get-Disk
Get-Partition
Get-Volume
```

Expected:

- Disk = 30 GB
- C: = 30 GB

---

# Step 8 - Increase EBS Volume

AWS Console

EC2 → Volumes

Select the **root volume** attached to the private EC2.

Actions → Modify Volume

Old Size:
```
30 GB
```

New Size:
```
50 GB
```

Click **Modify**.

No reboot required.

---

# Step 9 - Wait for Modification

State changes:

- Modifying
- Optimizing
- Completed

---

# Step 10 - Verify Disk

```powershell
Get-Disk
```

Disk Size:
```
30 GB
```

Partition still:

```
20 GB
```

---

# Step 11 - Extend C Drive

GUI:

```
diskmgmt.msc
```

Right Click C:

→ Extend Volume

Finish.



---

# Step 12 - Verify

```powershell
Get-Volume
```

Expected:

C: = 50 GB

---

# Validation

- Bastion reachable from Internet
- Private EC2 reachable only through Bastion
- Root EBS increased from 30 GB to 50 GB
- Windows C: drive extended successfully
- No downtime
- Data preserved

---

# Troubleshooting

## Cannot RDP to Private EC2

- Verify Bastion can reach the private IP.
- Confirm Private EC2 SG allows TCP 3389 from Bastion SG.

## Extend Volume Disabled

Ensure there is contiguous unallocated space after C:.

## Disk still 30 GB

Run:

```powershell
Update-HostStorageCache
```

---

# Best Practices

- Restrict Bastion RDP to your IP.
- Use snapshots before resizing production volumes.
- Delete Bastion when the maintenance task is complete.

---

# Interview Questions

**Why use a Bastion Host?**

To securely administer instances in private subnets without assigning them public IP addresses.

**Does modifying an EBS volume require downtime?**

No.

**Why is extending the Windows partition necessary?**

AWS increases the virtual disk size; Windows must extend the filesystem separately.

---

# Cleanup

1. Disconnect RDP sessions.
2. Terminate Bastion Host if no longer needed.
3. Terminate Private EC2 (if this is a lab).
4. Delete unused EBS volumes.
5. Remove temporary security group rules.

---

