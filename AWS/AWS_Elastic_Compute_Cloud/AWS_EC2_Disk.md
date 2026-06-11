# AWS Hands-on Lab - Decrease EC2 Disk Size by 10 GB 

# Objective

Reduce the root disk size of an EC2 instance from **30 GB to 20 GB**.

Since AWS does not support shrinking EBS volumes directly, we will:

* Create an AMI
* Launch a temporary EC2 with a smaller root volume
* Verify data
* Replace the old instance

This is the safest and recommended production approach.

---

# Scenario

Current EC2

```
Instance Name

Production-Server

Instance Type

t3.medium

Root Volume

30 GB

Used Space

8 GB
```

Requirement

```
Reduce Root Volume

30 GB

↓

20 GB
```

---

# Why Can't AWS Shrink an EBS Volume?

AWS allows:

```
20 GB

↓

30 GB

↓

50 GB

↓

100 GB
```

But not:

```
100 GB

↓

50 GB

❌
```

Shrinking could cause filesystem corruption if data exists beyond the new size.

Therefore, AWS only supports volume expansion.

---

# Lab Architecture

```
Old EC2 (30 GB)
        │
        │
Create AMI
        │
        ▼
Amazon Machine Image
        │
Launch New EC2
with 20 GB root disk
        │
        ▼
Verify Application
        │
        ▼
Update Elastic IP / DNS
        │
        ▼
Terminate Old EC2
```

---

# Step 1 - Check Current Disk Usage

Connect to EC2

```
ssh -i key.pem ec2-user@PUBLIC_IP
```

Check disks

```
lsblk
```

Example

```
NAME      SIZE

xvda       30G

└─xvda1    30G
```

---

Check filesystem

```
df -h
```

Example

```
Filesystem

/dev/xvda1

Size

30G

Used

8G

Available

22G
```

Verify that used space is less than the new target size.

---

# Step 2 - Stop Applications

Example

```
sudo systemctl stop nginx

sudo systemctl stop docker

sudo systemctl stop httpd
```

or

```
docker stop $(docker ps -q)
```

This ensures a consistent image.

---

# Step 3 - Create AMI

AWS Console

```
EC2

↓

Instances

↓

Select Instance

↓

Actions

↓

Image and Templates

↓

Create Image
```

Name

```
production-backup-ami
```

Click

```
Create Image
```

Wait until the AMI status becomes

```
Available
```

---

# Step 4 - Launch New EC2 from AMI

Go to

```
AMIs

↓

Select production-backup-ami

↓

Launch Instance from AMI
```

---

# Step 5 - Modify Root Volume Size

Storage section

Current

```
30 GB
```

Change to

```
20 GB
```

AWS validates that the image fits into the smaller volume.

If the used data exceeds 20 GB, the launch will fail.

---

Instance Type

```
t3.medium
```

Security Group

Use the same as the old server.

Key Pair

Use the existing key.

Launch instance.

---

# Step 6 - Connect to New Server

```
ssh -i key.pem ec2-user@NEW_PUBLIC_IP
```

---

# Step 7 - Verify Disk Size

```
lsblk
```

Expected

```
NAME

xvda

20G
```

---

Check filesystem

```
df -h
```

Expected

```
Filesystem

/dev/xvda1

Size

20G
```

---

# Step 8 - Verify Data

Check files

```
ls

pwd
```

Verify application files

```
cd /var/www/html

ls
```

or

```
cd /opt

ls
```

---

If Docker is installed

```
docker ps -a

docker images
```

Verify containers and images.

---

# Step 9 - Start Services

```
sudo systemctl start nginx

sudo systemctl start docker

sudo systemctl start httpd
```

Verify

```
systemctl status nginx

systemctl status docker
```

---

# Step 10 - Verify Application

Browser

```
http://PUBLIC_IP
```

or

```
curl localhost
```

Check logs

```
journalctl -xe

docker logs CONTAINER_ID
```

Everything should work exactly as before.

---

# Step 11 - Switch Traffic

## Option 1 - Elastic IP

Detach Elastic IP from old instance.

Attach Elastic IP to new instance.

No DNS changes required.

---

## Option 2 - Route53

Update A record

```
Old EC2

↓

New EC2
```

Wait for DNS propagation.

---

## Option 3 - Load Balancer

```
Application Load Balancer

↓

Remove Old Target

↓

Add New Target

↓

Health Check

↓

Route Traffic
```

This is the preferred production method.

---

# Step 12 - Validate

Check

```
curl localhost

docker ps

df -h

free -m

top
```

Verify

* Application
* Logs
* CPU
* Memory
* Disk
* Networking

---

# Step 13 - Terminate Old Instance

After validation

```
EC2

↓

Old Instance

↓

Terminate
```

Delete old snapshots and AMIs if no longer required.

---

# AWS CLI Commands

## List Volumes

```bash
aws ec2 describe-volumes
```

---

## List Snapshots

```bash
aws ec2 describe-snapshots --owner-ids self
```

---

## List AMIs

```bash
aws ec2 describe-images --owners self
```

---

## Describe Instance

```bash
aws ec2 describe-instances
```

---

## Check Attached Volume

```bash
aws ec2 describe-volumes \
--filters Name=attachment.instance-id,Values=i-xxxxxxxx
```

---

# Linux Commands

Check block devices

```bash
lsblk
```

Disk usage

```bash
df -h
```

Filesystem UUID

```bash
blkid
```

Partitions

```bash
fdisk -l
```

Filesystem type

```bash
file -s /dev/xvda1
```

Memory

```bash
free -m
```

CPU

```bash
lscpu
```

Processes

```bash
top
```

---

# Common Mistakes

## Mistake

Trying to modify EBS from

```
30 GB

↓

20 GB
```

Result

```
Not Supported
```

---

## Mistake

Creating a 20 GB disk while 25 GB is already used.

Result

```
Instance launch or migration will fail.
```

---

## Mistake

Deleting old EC2 before testing the new one.

Always keep the old instance until validation is complete.

---

# Interview Questions

## Can you decrease an EBS volume size?

**Answer:**

No. AWS supports increasing EBS volume size but does not support shrinking it directly.

---

## How do you reduce the root volume size of an EC2 instance?

Create an AMI (or backup), launch a new instance with a smaller root volume if the data fits, verify the application, then redirect traffic and retire the old instance.

---

## Why doesn't AWS support shrinking EBS volumes?

Shrinking can corrupt the filesystem if blocks beyond the new boundary contain data. Expanding is safe; shrinking requires migration.

---

# Real DevOps Workflow

```
Production EC2 (30 GB)
          │
          ▼
      Create AMI
          │
          ▼
Launch New EC2 (20 GB)
          │
          ▼
Functional Testing
          │
          ▼
Health Checks
          │
          ▼
Switch Traffic
          │
          ▼
Monitor
          │
          ▼
Terminate Old EC2
```

---

# Final Learning Outcome

After completing this lab, you will understand:

* Why AWS EBS volumes cannot be shrunk directly
* How to safely reduce EC2 disk size
* How to use AMIs for migration
* How to validate applications after migration
* How to perform production cutover using Elastic IP, Route 53, or an Application Load Balancer
* Best practices followed by DevOps engineers for storage migration with minimal downtime
