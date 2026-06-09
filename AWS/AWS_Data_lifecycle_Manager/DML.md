
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/609fcd76-d244-4a48-99f7-20e0123afce6" />


---

# Introduction

Data protection is one of the most important responsibilities of a DevOps or Cloud Engineer.

Imagine a production EC2 server crashes because of:

* Human error
* Accidental file deletion
* OS corruption
* Malware
* Failed software upgrade

Without backups, all data may be permanently lost.

AWS provides **Amazon Data Lifecycle Manager (DLM)** to automate EBS snapshots so backups happen automatically without manual effort.

---

# What is Data Lifecycle Manager (DLM)?

Amazon Data Lifecycle Manager (DLM) is an AWS service that automates the creation, retention, and deletion of Amazon EBS snapshots based on schedules.

Instead of manually creating snapshots every day, DLM performs this task automatically according to the lifecycle policy you configure.

---

# What does DLM do?

DLM can automatically:

* Create snapshots
* Delete old snapshots
* Create backups on a schedule
* Manage retention periods
* Reduce manual operational work

---

# Why Do We Need DLM?

## Without DLM

```
Day 1
Create Snapshot manually

Day 2
Create Snapshot manually

Day 3
Forget to create backup ❌

Server crashes

No latest backup available
```

Manual backups are unreliable.

---

## With DLM

```
Every Day 2:00 AM

↓

Snapshot Created Automatically

↓

Older Snapshots Deleted Automatically

↓

No Manual Work
```

Backups happen consistently without administrator intervention.

---

# Benefits of DLM

## Automated Backups

No need to create snapshots manually.

---

## Cost Optimization

Old snapshots are automatically deleted based on the retention policy.

No unnecessary storage costs.

---

## Disaster Recovery

If a server or volume fails, you can restore from a snapshot.

---

## Data Protection

Protects against accidental deletion or corruption.

---

## Tag-Based Management

One policy can back up hundreds of EC2 instances using tags.

Example:

```
Backup=Daily
```

Every tagged instance is automatically protected.

---

## Scalable

Works for single servers as well as enterprise environments.

---

# How DLM Works

```
EC2 Instance
       │
       ▼
EBS Volume
       │
       ▼
Tag (Backup=Daily)
       │
       ▼
Data Lifecycle Manager
       │
       ▼
Daily Schedule
       │
       ▼
Create Snapshot
       │
       ▼
Retention Policy
       │
       ▼
Delete Old Snapshots
```

---

# Complete Workflow

```
                EC2 Instance
                      │
             Root EBS Volume
                      │
                      ▼
           Tag = Backup:Daily
                      │
                      ▼
        Data Lifecycle Manager Policy
                      │
                      ▼
       Daily at 2:00 AM IST
      (20:30 UTC Previous Day)
                      │
                      ▼
        Create EBS Snapshot
                      │
                      ▼
       Keep Latest 2 Snapshots
                      │
                      ▼
Delete Older Snapshots Automatically
```

---

# Prerequisites

Before creating a DLM policy:

* AWS Account
* Running EC2 Instance
* EBS Volume attached
* IAM permissions
* EC2 properly tagged

---

# Hands-on Practical

## Step 1: Launch EC2

Launch an EC2 instance.

Example:

```
Name

web-server
```

Verify that an EBS volume is attached.

---

## Step 2: Add Tag  ( its important to add because you can take a backup of specific instance or volumes )  

Go to:

```
EC2

↓

Instances

↓

Select Instance

↓

Tags

↓

Manage Tags
```

Create:

| Key    | Value |
| ------ | ----- |
| Backup | Daily |

### Why?

DLM identifies which resources to back up using tags.

---

## Step 3: Open Lifecycle Manager

Navigate to:

```
AWS Console

↓

EC2

↓

Elastic Block Store

↓

Lifecycle Manager
```

Click

```
Create Lifecycle Policy
```

---

## Step 4: Select Policy Type

Choose

```
EBS Snapshot Policy
```

### Why?

We want to automate EBS snapshots.

---

## Step 5: Target Resources
- If you selects Instance it will take backup of whole EC2 with attached volumes
- if you selects Volume it will  take backup of volume only 
Choose

```
Target Resource Type

Instance
```

Select by tag:

```
Key

Backup

Value

Daily
```

### Why?

Every EC2 with this tag will automatically be included.

---

## Step 6: Configure Schedule

| Setting        | Value            |
| -------------- | ---------------- |
| Schedule Name  | Daily-2AM-Backup |
| Frequency      | Daily            |
| Every          | 24 Hours         |
| Starting At    | 20:30 UTC        |
| Retention Type | Count            |
| Keep           | 2 Snapshots      |

---

## Why 20:30 UTC?

AWS uses UTC.

```
IST = UTC + 5:30

2:00 AM IST

↓

20:30 UTC
```

Configure:

```
20:30 UTC
```

to get snapshots at **2:00 AM IST**.

---

## Why Keep = 2?

Example:

```
Monday

Snapshot-1

Tuesday

Snapshot-2

Wednesday

Snapshot-3

Snapshot-1 Deleted

Thursday

Snapshot-4

Snapshot-2 Deleted
```

Only the latest two snapshots remain.

---

## Step 7: Review

Verify:

* Correct tag
* Daily schedule
* 20:30 UTC
* Count = 2

Click

```
Create Policy
```

The policy becomes active.

---

# Verification

After the scheduled time:

Go to

```
EC2

↓

Snapshots
```

You should see:

```
Snapshot-01

Completed
```

Next day:

```
Snapshot-02

Completed
```

Third day:

```
Snapshot-03

Completed

Snapshot-01 Deleted
```

Retention is working correctly.

---

# Restore Process

Suppose your EBS volume becomes corrupted.

Go to:

```
EC2

↓

Snapshots

↓

Select Snapshot

↓

Create Volume

↓

Attach Volume
```

Or launch a new EC2 using the restored volume.

Recovery is fast because snapshots are stored in Amazon S3 internally.

---

# Best Practices

* Use tag-based policies.
* Keep at least 7 daily backups for production workloads.
* Test restore procedures regularly.
* Enable encryption for EBS volumes.
* Document backup and recovery processes.
* Monitor snapshot creation through CloudWatch and EventBridge.
* Keep backups in another region for disaster recovery if required.

---

# DLM vs Manual Backup

| Manual                  | DLM               |
| ----------------------- | ----------------- |
| Manual work             | Automated         |
| Easy to forget          | Scheduled         |
| No retention            | Automatic cleanup |
| Time consuming          | Zero maintenance  |
| Higher operational risk | Reliable          |

---

# Real-World Example

A production web server hosts a business-critical application.

The DevOps team adds:

```
Backup=Daily
```

They create a DLM policy:

* Frequency: Daily
* Time: 2:00 AM IST
* Retention: 2 snapshots

Every day DLM creates a new EBS snapshot and automatically deletes the oldest snapshot after the retention limit is reached.

If the server crashes, the latest snapshot can be restored to recover the application.

---

# Interview Questions

## What is DLM?

Amazon Data Lifecycle Manager is an AWS service that automates EBS snapshot creation, retention, and deletion using lifecycle policies.

---

## Why use DLM?

To automate backups, reduce manual work, lower storage costs, and improve disaster recovery.

---

## Does DLM back up EC2 or EBS?

DLM creates **EBS snapshots**. Since EC2 data resides on EBS volumes, those volumes are what get backed up.

---

## Can DLM delete old snapshots?

Yes. Based on the configured retention policy, DLM automatically deletes old snapshots.

---

## How does DLM identify resources?

Using **tags** such as:

```
Backup=Daily
```

---

## Can DLM restore an EC2 instance?

No.

DLM only creates snapshots.

To restore, create a new EBS volume from the snapshot and attach it to an EC2 instance or launch a new server using the recovered data.

---

# Conclusion

Amazon Data Lifecycle Manager is a simple, reliable, and cost-effective service for automating EBS backups.

By using lifecycle policies, scheduled snapshots, and automatic retention management, organizations can protect critical workloads while minimizing operational overhead and storage costs. It is a fundamental service every AWS and DevOps engineer should understand for backup and disaster recovery strategies.
