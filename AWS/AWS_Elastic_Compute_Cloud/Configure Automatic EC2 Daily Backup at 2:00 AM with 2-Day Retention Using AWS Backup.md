# Configure Automatic EC2 Daily Backup at 2:00 AM with 2-Day Retention Using AWS Backup

---

# Lab Objective

In this lab, you will configure automatic backups for an EC2 instance using AWS Backup.

At the end of this lab, you will have:

* Daily backup scheduled at **2:00 AM**
* Automatic snapshot creation
* Backup vault for storing backups
* IAM service role for AWS Backup
* Backup plan
* Resource assignment
* Lifecycle policy to automatically delete backups after **2 days**
* Ability to restore the EC2 instance if required

---

# Why Do We Need EC2 Backups?

Production servers can fail due to:

* Human mistakes
* OS corruption
* Malware or ransomware
* Application failure
* Data corruption
* Accidental deletion
* AWS infrastructure issues

A backup allows you to restore the server to a previous working state with minimal downtime.

---

# What is AWS Backup?

AWS Backup is a fully managed service that centralizes and automates data protection across AWS services.

It supports:

* EC2
* EBS
* RDS
* DynamoDB
* EFS
* S3


Instead of manually creating snapshots every day, AWS Backup automatically creates and manages them based on a schedule.

---

# Architecture

```
EC2 Instance

        │

        ▼

AWS Backup Plan

        │

        ▼

Backup Rule

(Daily at 2 AM)

        │

        ▼

Backup Vault

        │

        ▼

EBS Snapshots

        │

        ▼

Retention Policy

(Delete after 2 Days)
```

---

# Prerequisites

* AWS Account
* Existing EC2 Instance
* Required IAM permissions
* Region selected

---

# Step 1 — Login to AWS Console

Open

```
https://console.aws.amazon.com
```

Login with your AWS account.

---

# Step 2 — Open AWS Backup

Search

```
AWS Backup
```

Open the service.

---

# Step 3 — Create Backup Vault

## What is a Backup Vault?

A Backup Vault is a secure container where AWS stores backups.

Think of it as a folder for snapshots.

---

Click

```
Backup vaults

Create Backup Vault
```

Fill:

```
Backup Vault Name

EC2-Daily-Backup-Vault
```

Encryption Key

```
Default AWS Managed KMS Key
```

Click

```
Create Backup Vault
```

---

# Step 4 — Create Backup Plan

Go to

```
Backup Plans

Create Backup Plan
```

Select

```
Build a new plan
```

Click

```
Create
```

---

# Step 5 — Configure Backup Plan

Backup Plan Name

```
EC2-Daily-Backup
```

---

# Step 6 — Configure Backup Rule

Rule Name

```
Daily-2AM
```

---

# Backup Vault

Select

```
EC2-Daily-Backup-Vault
```

---

# Backup Frequency

Choose

```
Daily
```

---

# Backup Window

Start Window

```
60 Minutes
```

Completion Window

```
180 Minutes
```

---

# Step 7 — Configure Backup Schedule

Choose

```
Cron Expression
```

Enter

```
cron(30 20 * * ? *)
```

---

## Why cron(30 20 * * ? *)?

AWS Backup schedules are based on UTC.

India Standard Time (IST)

```
UTC +5:30
```

Desired backup time

```
2:00 AM IST
```

Equivalent UTC

```
8:30 PM UTC (Previous Day)
```

Cron

```
cron(30 20 * * ? *)
```

Result:

Every day at **2:00 AM IST**.

---

# Step 8 — Configure Lifecycle

Enable Lifecycle.

Set

```
Delete After

2 Days
```

This means:

```
Day 1 Backup

↓

Day 2 Backup

↓

Day 3

Day 1 Backup Automatically Deleted
```

Only the last two days of backups will be retained.

---

# Step 9 — Create Backup Plan

Click

```
Create Plan
```

Wait until the backup plan is created.

---

# Step 10 — Assign Resources

Open

```
Backup Plan

Assign Resources
```

---

# Resource Assignment Name

```
EC2-Production
```

---

# IAM Role

Choose

```
Default Role

AWSBackupDefaultServiceRole
```

If it does not exist, AWS will create it automatically.

---

# Resource Type

Select

```
EC2
```

---

# Choose Resources

Select your EC2 instance.

Example

```
i-0123456789abcdef
```

Click

```
Assign Resources
```

---

# Step 11 — Verify Assignment

Open

```
Protected Resources
```

You should see your EC2 instance listed.

Status

```
Protected
```

---

# Step 12 — Run an On-Demand Backup (Optional)

To verify the configuration immediately:

Select the EC2 resource.

Click

```
Create On-demand Backup
```

Choose

```
EC2-Daily-Backup-Vault
```

Click

```
Create Backup
```

Wait until the job status changes to:

```
Completed
```

---

# Step 13 — Verify Backup Job

Go to

```
Backup Jobs
```

You should see:

```
Status

Completed
```

Backup Type

```
Scheduled
```

or

```
On-demand
```

---

# Step 14 — Verify Recovery Points

Go to

```
Backup Vault

EC2-Daily-Backup-Vault
```

You should see recovery points (snapshots) created for the EC2 instance.

---

# Step 15 — Restore the EC2 Instance (If Needed)

Go to

```
Backup Vault

Recovery Point
```

Click

```
Restore
```

Choose:

* Instance type
* VPC
* Subnet
* Security Group
* IAM Role
* Key Pair

Click

```
Restore Backup
```

AWS will create a new EC2 instance from the backup.

---

# Monitoring

Navigate to

```
AWS Backup

Backup Jobs
```

Monitor:

* Running jobs
* Completed jobs
* Failed jobs
* Recovery points

---

# Best Practices

* Use a dedicated Backup Vault.
* Enable encryption using AWS KMS.
* Test restore procedures regularly.
* Monitor backup failures.
* Tag resources consistently.
* Use least-privilege IAM roles.
* Configure backup notifications with Amazon SNS.
* For production, enable cross-region or cross-account backup copies.
* Periodically verify recovery points and restore integrity.

---

# Cost Considerations

AWS Backup charges for:

* Backup storage
* Cross-region copies (if enabled)
* Restore operations

With a 2-day retention policy, storage costs remain relatively low because older recovery points are automatically deleted.

---

# Workflow Summary

```
EC2 Instance

        │

        ▼

AWS Backup Plan

        │

        ▼

Daily Schedule

(2:00 AM IST)

        │

        ▼

AWS Backup Service

        │

        ▼

Backup Vault

        │

        ▼

Recovery Point

        │

        ▼

Retain for 2 Days

        │

        ▼

Automatically Deleted
```

---
