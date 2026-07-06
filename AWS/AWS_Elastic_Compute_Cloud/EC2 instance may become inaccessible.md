# Amazon EC2 Troubleshooting Guide

## Objective

This guide covers **every common reason** why an EC2 instance may become inaccessible and provides a **step-by-step troubleshooting process**. It is designed for AWS Cloud Engineers, DevOps Engineers, and System Administrators.

---

# Troubleshooting Flow

```

Can't Access EC2
│
├── Is the EC2 Running?
│      │
│      ├── No → Start the Instance
│      └── Yes
│
├── Status Checks Passed?
│      │
│      ├── Failed → Identify Failed Check
│      └── Passed
│
├── Public IP / Elastic IP Correct?
│      │
│      ├── No → Update IP
│      └── Yes
│
├── Security Group Allows SSH/RDP?
│      │
│      ├── No → Update SG
│      └── Yes
│
├── NACL Blocking?
│      │
│      ├── Yes → Fix NACL
│      └── No
│
├── Route Table Correct?
│      │
│      ├── No → Fix Route
│      └── Yes
│
├── Internet Gateway Attached?
│      │
│      ├── No → Attach IGW
│      └── Yes
│
├── SSH Key Correct?
│      │
│      ├── No → Recover Instance
│      └── Yes
│
├── SSH Service Running?
│      │
│      ├── No → Restart SSH
│      └── Yes
│
├── Disk Full?
│      │
│      ├── Yes → Free Space
│      └── No
│
├── CPU or Memory Hung?
│      │
│      ├── Yes → Reboot
│      └── No
│
└── Hardware Issue?
       │
       ├── Yes → Stop/Start
       └── Contact AWS
```

---

# 1. EC2 Instance is Stopped

## Symptoms

```
Connection timed out
```

or

```
Host unreachable
```

## Verify

```
EC2 Console
↓

Instance State
```

Should be

```
Running
```

## Solution

Start the instance.

---

# 2. Status Checks Failed

Open

```
EC2
↓

Status Checks
```

Possible:

```
3/3 Passed
```

or

```
2/3 Passed
```

or

```
Failed
```

---

## A. System Status Failed

Reason

- AWS hardware issue
- Physical host failure
- Hypervisor issue
- AWS network issue

Solution

- Wait
- Stop/Start instance
- Contact AWS

---

## B. Instance Status Failed

Reason

- Linux failed boot
- Kernel panic
- Bad fstab
- File system corruption
- Disk full

Solution

- EC2 Serial Console
- Systems Manager
- Recover root volume

---

## C. Attached EBS Failed

Reason

Storage problem

Solution

Restore from Snapshot

---

# 3. Wrong Public IP

Public IP changes after:

```
Stop
↓

Start
```

unless using Elastic IP.

Check

```
EC2

↓

Public IPv4
```

Update SSH command.

---

# 4. Elastic IP Not Associated

Verify

```
Elastic IP

↓

Associated?
```

Associate it.

---

# 5. Wrong SSH Key

Error

```
Permission denied (publickey)
```

Reason

Wrong PEM file

Wrong Key Pair

Solution

Recover instance

Attach volume

Replace authorized_keys

---

# 6. Wrong File Permission

PEM should be

```
chmod 400 key.pem
```

Otherwise

```
Permissions 0644 are too open
```

---

# 7. Wrong Username

Amazon Linux

```
ec2-user
```

Ubuntu

```
ubuntu
```

RHEL

```
ec2-user
```

Debian

```
admin
```

CentOS

```
centos
```

---

# 8. Security Group Blocking SSH

Inbound Rule

```
22

TCP

Your IP
```

or

```
3389

Windows
```

Verify

```
EC2

↓

Security Group

↓

Inbound Rules
```

---

# 9. Your IP Changed

ISP changes IP

Security Group allows old IP

Update

```
x.x.x.x/32
```

---

# 10. Network ACL Blocking

Inbound

```
22 Allow
```

Outbound

```
1024-65535 Allow
```

Remember

NACL is Stateless.

---

# 11. Route Table Missing

Public subnet needs

```
0.0.0.0/0

↓

Internet Gateway
```

Private subnet needs

```
0.0.0.0/0

↓

NAT Gateway
```

---

# 12. Internet Gateway Missing

Verify

```
VPC

↓

Internet Gateway
```

Attached?

If not

Attach IGW

---

# 13. Instance in Private Subnet

No Public IP

Cannot SSH directly.

Solutions

- Bastion Host
- SSM
- VPN
- Pritunl
- AWS Client VPN

---

# 14. SSH Service Stopped

Verify

```
systemctl status sshd
```

Restart

```
sudo systemctl restart sshd
```

---

# 15. SSH Port Changed

Config

```
/etc/ssh/sshd_config
```

Example

```
Port 2222
```

Connect

```
ssh -p 2222
```

---

# 16. Firewall Blocking

Ubuntu

```
ufw status
```

RHEL

```
firewalld
```

Allow SSH.

---

# 17. Disk Full

```
df -h
```

Symptoms

Cannot login

Cannot create session

Solution

Delete logs

Increase EBS

---

# 18. CPU 100%

Symptoms

SSH timeout

Check

```
top
```

or

```
htop
```

---

# 19. Memory Exhausted

OOM Killer

High RAM

SSH hangs

Solution

Reboot

Increase RAM

---

# 20. File System Corruption

Repair

```
fsck
```

Usually by attaching root volume to another instance.

---

# 21. Wrong DNS

Use

```
Public IPv4
```

instead of DNS.

---

# 22. IMDS Disabled (Application Issue)

Usually affects applications, not SSH.

Verify

```
IMDSv2
```

---

# 23. VPC Peering Issue

Remote VPC cannot reach instance.

Verify

- Route Tables
- SG
- NACL

---

# 24. VPN Down

If EC2 is private

Verify

- VPN Tunnel
- Pritunl
- Client VPN

---

# 25. Bastion Host Down

Cannot reach private EC2.

Check Bastion first.

---

# 26. NAT Gateway Failure

Private EC2

No Internet

Cannot install packages

---

# 27. AWS Systems Manager Not Configured

Cannot use Session Manager.

Need

- IAM Role
- SSM Agent
- VPC Endpoints or Internet

---

# 28. EBS Detached

Root volume detached.

Instance won't boot.

---

# 29. Wrong AMI

Boot issues

Kernel mismatch

---

# 30. Corrupted Boot Loader

GRUB problem

Repair root volume.

---

# 31. Kernel Panic

Symptoms

Instance check failed

Serial Console shows panic

---

# 32. ENI Misconfiguration

Wrong Network Interface

Wrong Security Group

Wrong Subnet

---

# 33. Source/Destination Check

Required

```
Disabled
```

for

- NAT Instance
- Firewall Appliance

---

# 34. Elastic Network Interface Removed

Primary ENI missing

Instance inaccessible.

---

# 35. Wrong MTU

VPN issues

Packets dropped.

---

# 36. DNS Resolution Disabled in VPC

Verify

```
enableDnsSupport

enableDnsHostnames
```

---

# 37. IAM Issue

Cannot connect using

```
SSM
```

because IAM Role missing.

---

# 38. SSH Brute Force Protection

Fail2Ban

Blocks your IP.

---

# 39. SELinux Blocking

Check

```
getenforce
```

---

# 40. AWS Maintenance Event

Check

```
EC2

↓

Scheduled Events
```

---

# Useful AWS CLI Commands

## Instance Status

```bash
aws ec2 describe-instance-status --instance-ids i-xxxxxxxx
```

---

## Security Groups

```bash
aws ec2 describe-security-groups
```

---

## Route Tables

```bash
aws ec2 describe-route-tables
```

---

## Network ACL

```bash
aws ec2 describe-network-acls
```

---

## Elastic IP

```bash
aws ec2 describe-addresses
```

---

## ENI

```bash
aws ec2 describe-network-interfaces
```

---

# Linux Commands

Disk

```bash
df -h
```

Memory

```bash
free -h
```

CPU

```bash
top
```

SSH

```bash
systemctl status sshd
```

Logs

```bash
journalctl -xe
```

Kernel

```bash
dmesg
```

---

# Best Practice Checklist

- Use Elastic IP for public servers.
- Restrict SSH access in Security Groups.
- Keep SSH keys secure (`chmod 400`).
- Enable EC2 Status Check monitoring with CloudWatch.
- Configure AWS Systems Manager for emergency access.
- Regularly create EBS snapshots.
- Monitor CPU, memory, and disk usage.
- Use Auto Scaling for production workloads.
- Enable VPC Flow Logs and CloudTrail for troubleshooting.
- Test disaster recovery procedures periodically.

---

# Conclusion

When an EC2 instance is inaccessible, always troubleshoot from the **outside in**:

1. Verify the instance state.
2. Check EC2 status checks (System, Instance, EBS).
3. Confirm networking (Public IP, IGW, Route Table, NACL, Security Group).
4. Validate SSH/RDP configuration (key pair, username, service, firewall).
5. Inspect operating system health (CPU, memory, disk, filesystem, logs).
6. Review advanced issues (ENI, VPN, IAM, SSM, maintenance events).

Following this structured approach helps identify and resolve the vast majority of EC2 connectivity issues efficiently.
