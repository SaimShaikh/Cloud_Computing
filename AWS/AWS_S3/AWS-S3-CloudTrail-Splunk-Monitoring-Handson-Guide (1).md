# AWS S3 Object-Level Activity Monitoring with CloudTrail, SQS & Splunk
## Complete Hands-On Production-Style Lab Guide (Demo/POC Build)

**Author:** Saime Shaikh
**Status:** Final — ready for team walkthrough / demo
**Version:** 1.0 (2026-07-03)
**Project Type:** Cloud Security / DevOps Monitoring Project
**Scope:** Demo / POC build (not the production-hardened variant — see Section 14.1 for what changes in production)
**Difficulty:** Beginner → Intermediate → Enterprise Perspective
**Stack:** Amazon S3, AWS CloudTrail (Data Events), Amazon SQS, IAM, EC2, Splunk Enterprise, Splunk Add-on for AWS

> This guide follows a strict "no skipped steps" hands-on format. Every AWS console click, every CLI command, every IAM policy, and every Splunk configuration screen is documented in order. Follow it top to bottom exactly once for the demo build, then re-read the "Enterprise Perspective" call-outs to understand how this scales to production.
>
> **Before presenting to the team:** replace every `<yourname>`, `<ACCOUNT_ID>`, `<ACCESS_KEY_ID>`, and `<INSTANCE_ID>` placeholder with your real values, or leave them as-is if this is being shared as a reusable team template (recommended for a shared doc).

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Prerequisites](#2-prerequisites)
3. [Core Concepts Explained From First Principles](#3-core-concepts-explained-from-first-principles)
4. [Architecture](#4-architecture)
5. [Hands-On Lab Part 1 — S3 Buckets Foundation](#5-hands-on-lab-part-1--s3-buckets-foundation)
6. [Hands-On Lab Part 2 — CloudTrail Configuration](#6-hands-on-lab-part-2--cloudtrail-configuration)
7. [Hands-On Lab Part 3 — Amazon SQS + Event Notification Wiring](#7-hands-on-lab-part-3--amazon-sqs--event-notification-wiring)
8. [Hands-On Lab Part 4 — IAM for Splunk](#8-hands-on-lab-part-4--iam-for-splunk)
9. [Hands-On Lab Part 5 — EC2 + Splunk Enterprise Installation](#9-hands-on-lab-part-5--ec2--splunk-enterprise-installation)
10. [Hands-On Lab Part 6 — Splunk Add-on for AWS Configuration](#10-hands-on-lab-part-6--splunk-add-on-for-aws-configuration)
11. [End-to-End Testing & Verification](#11-end-to-end-testing--verification)
12. [Splunk SPL Searches & Dashboard Build](#12-splunk-spl-searches--dashboard-build)
13. [Alerts, Troubleshooting & Failure Scenarios](#13-alerts-troubleshooting--failure-scenarios)
14. [Best Practices, Cost, Cleanup & Interview Cheat Sheet](#14-best-practices-cost-cleanup--interview-cheat-sheet)

---

## 1. Project Overview

### 1.1 What is this project?

This project builds an **object-level audit and monitoring pipeline** for Amazon S3. Every time someone uploads (`PutObject`), downloads (`GetObject`), or deletes (`DeleteObject`) a file in a monitored S3 bucket, that action is captured, transported, and displayed as a searchable, dashboard-ready event inside **Splunk**.

In plain language: *"Every time a file touches this S3 bucket, I want to see who did it, from where, and when — inside Splunk, in near real time."*

### 1.2 Why is it needed?

S3 does **not** keep a human-readable "who uploaded what" log by default. S3 Server Access Logging exists, but it is delayed, file-based, and painful to query. Enterprises need:

- **Audit trails** — regulatory frameworks (PCI-DSS, HIPAA, SOC 2, ISO 27001, RBI/IRDAI in India) require proof of who accessed sensitive data and when.
- **Security monitoring** — detect anomalous access (e.g., mass `DeleteObject` calls, access from unexpected IP ranges, access outside business hours).
- **Operational visibility** — confirm that automated pipelines (backups, ETL jobs, uploads from partner systems) are actually running.
- **Incident response** — when something goes wrong (data leak, ransomware-style deletion), you need a searchable timeline, not raw JSON files scattered across S3.

### 1.3 What business problem does it solve?

| Business Problem | How This Project Solves It |
|---|---|
| "We don't know who deleted that file" | Every `DeleteObject` call is captured with IAM identity, source IP, and timestamp |
| "Compliance audit needs 1 year of S3 access history" | CloudTrail + Splunk gives a durable, searchable audit trail |
| "Security team has no visibility into S3 activity" | Splunk dashboards give real-time visualizations of uploads/downloads/deletes |
| "Too many raw JSON log files to search manually" | Splunk indexes and parses fields automatically — searchable in seconds |
| "No alerting when sensitive data is accessed" | Splunk alerts trigger on suspicious patterns |

### 1.4 Real-World Use Cases

- **Fintech**: Monitoring access to a bucket holding KYC documents / bank statements.
- **Healthcare**: HIPAA-driven audit trail for a bucket holding patient records.
- **SaaS product**: Monitoring a customer-uploads bucket to detect abuse (e.g., malware uploads, scraping).
- **DevOps**: Watching a CI/CD artifact bucket to confirm build artifacts are published correctly and not tampered with.
- **Security Operations Center (SOC)**: Feeding S3 activity into a broader SIEM (Splunk) alongside VPC Flow Logs, GuardDuty, and WAF logs for correlation.

### 1.5 Expected Outcome of This Lab

By the end of this guide, uploading a file to your demo S3 bucket will produce a searchable Splunk event within roughly 5–20 minutes, showing:

```
Event Name : PutObject
Bucket     : demo-upload-bucket-<yourname>
Object Key : resume.pdf
IAM Identity: arn:aws:iam::<account-id>:user/admin
Source IP  : 49.xx.xx.xx
Region     : ap-south-1
Timestamp  : 2026-07-03T10:15:00Z
```

### 1.6 Learning Objectives

- S3 object-level logging vs. bucket-level logging
- AWS CloudTrail Management Events vs. Data Events
- CloudTrail log file structure and delivery mechanics
- S3 Event Notifications (`s3:ObjectCreated:*`)
- Amazon SQS as a decoupling/buffering layer
- Why SNS is **not** usable for direct Splunk delivery
- IAM least-privilege policy design for a monitoring tool
- Splunk Enterprise installation and initial hardening
- Splunk Add-on for AWS (TA-aws) SQS-based S3 input
- Splunk Search Processing Language (SPL) fundamentals
- Splunk dashboard and alert creation
- End-to-end troubleshooting of a log pipeline

---

## 2. Prerequisites

### 2.1 Required Knowledge (Beginner Level is Fine)

- Basic AWS Console navigation
- Basic Linux command line (`cd`, `sudo`, `systemctl`, `curl`)
- Basic understanding of JSON
- Basic SSH usage

You do **not** need prior Splunk experience — this guide explains SPL from scratch.

### 2.2 Accounts to Create

| Account | Purpose | Cost |
|---|---|---|
| AWS Account | Hosts S3, CloudTrail, SQS, EC2, IAM | Free Tier eligible, but this lab uses paid resources (EC2 t3.medium) |
| Splunk.com account | Required to download Splunk Enterprise | Free |

### 2.3 Software / Tools to Install Locally

- AWS CLI v2 (`aws --version` should show `aws-cli/2.x`)
- An SSH client (built-in on macOS/Linux; use PuTTY or Windows Terminal on Windows)
- A text editor (VS Code recommended)

### 2.4 Required AWS Services

- Amazon S3 (2 buckets: application bucket + CloudTrail log bucket)
- AWS CloudTrail
- Amazon SQS
- AWS IAM
- Amazon EC2 (to self-host Splunk Enterprise for this demo)

### 2.5 Cost Considerations (Important — Read Before You Start)

| Resource | Approx. Cost |
|---|---|
| EC2 `t3.medium` (Ubuntu 22.04, on-demand, `ap-south-1`) | ~$0.0416/hr (~₹3.5/hr) → **stop or terminate when not in use** |
| EBS 30 GB gp3 | ~$2.4/month if left running |
| S3 storage (demo files) | Negligible (a few cents) |
| CloudTrail Data Events | **First copy of management events is free; Data Events are billed per 100,000 events (~$0.10/100k)** — for a demo with a handful of uploads, cost is near-zero |
| SQS requests | First 1 million requests/month free |
| Splunk Enterprise | **Free for 60 days on a trial license (up to 500 MB/day indexing)** — perfect for this demo |

> **Enterprise Perspective:** In production, S3 Data Events on a high-traffic bucket (millions of `GetObject` calls) can get expensive fast. Production teams scope Data Events to specific prefixes/buckets, not the whole account, and often route through Kinesis Firehose instead of raw SQS polling for scale.

> **Cost Safety Tip:** Stop your EC2 instance (`aws ec2 stop-instances`) whenever you're not actively working through this lab. Terminate everything using Section 14 (Cleanup) once you're done demonstrating this project.

---

## 3. Core Concepts Explained From First Principles

### 3.1 Why doesn't S3 just log uploads by itself?

S3 is an object store, not a logging system. It optionally offers **S3 Server Access Logging**, which:
- Delivers logs with a delay of several hours
- Produces flat, space-delimited text files (not JSON) — hard to parse
- Does not include full IAM context in a structured way

This is why the industry-standard approach is **AWS CloudTrail Data Events**, which are near-real-time, structured JSON, and IAM-context-rich.

### 3.2 AWS CloudTrail — What, Why, When Not To Use

**What it is:** CloudTrail is AWS's API activity recorder. Every API call made against your AWS account (via Console, CLI, SDK) is logged as an event containing *who* called *what* API, *when*, from *where*, and with *what* parameters.

**Two categories of events:**

| Event Type | Example | Enabled by Default? |
|---|---|---|
| **Management Events** | `CreateBucket`, `RunInstances`, `PutBucketPolicy` | Yes (last 90 days, free, via Event History) |
| **Data Events** | `PutObject`, `GetObject`, `DeleteObject` on S3; `Invoke` on Lambda | **No — must be explicitly enabled**, and billed |

Object uploads/downloads/deletes are **Data Events**. This is the #1 concept people get wrong: *"CloudTrail already logs everything"* — false. Data Events must be turned on deliberately, per-bucket or per-prefix.

**Why companies use it:** It is the only AWS-native, tamper-evident (can be integrated with log file validation + S3 Object Lock) audit trail of API activity.

**When NOT to use it:** Don't enable Data Events on every bucket in the account blindly — on a bucket receiving millions of `GetObject` calls/day, this becomes a significant cost driver. Scope it.

### 3.3 S3 Event Notifications

**What it is:** A native S3 feature that fires a notification (to SNS, SQS, Lambda, or EventBridge) whenever something happens in a bucket — e.g., `s3:ObjectCreated:*`.

**Critical detail for this project:** We do **not** put the event notification on the *application* bucket where users upload files. We put it on the **CloudTrail log bucket** — because what we actually want Splunk to react to is *"a new CloudTrail log file has landed,"* not *"a user uploaded a file."* CloudTrail itself is what tells us a `PutObject` happened; S3 Event Notification is only the trigger that says *"go read the new CloudTrail log."*

### 3.4 Why SNS Cannot Be Used Directly With Splunk

This is a common point of confusion (and came up directly during the design of this project). Amazon SNS supports these subscriber types only:

- Email / Email-JSON
- SMS
- Amazon SQS
- AWS Lambda
- HTTP/HTTPS endpoints
- Mobile push (platform application endpoints)

**Splunk is not a native SNS subscriber type.** The Splunk Add-on for AWS knows how to poll an **SQS queue** and read S3 objects — it does not know how to receive an SNS push. So any "S3 → SNS → Splunk" design must actually be "S3 → SNS → SQS (subscribed to the SNS topic) → Splunk", or you bypass SNS entirely and go straight S3 → SQS. For a single-consumer demo like this one, **going straight to SQS is simpler and is what we build here.**

> **Enterprise Perspective:** SNS fan-out (S3 → SNS → multiple SQS queues) is used when *multiple* downstream systems need the same notification (e.g., Splunk **and** a Lambda-based automated remediation function **and** a data lake ingestion pipeline). For a single Splunk consumer, direct SQS is the leaner, recommended design — this is exactly the conclusion reached in this project's design discussion.

### 3.5 Amazon SQS — Why It's the Right Glue Here

| Property | Why It Matters for This Pipeline |
|---|---|
| **Durability** | Messages persist even if Splunk is temporarily down |
| **Retry** | Failed processing is retried automatically (visibility timeout) |
| **Buffering** | Absorbs bursts of CloudTrail log deliveries without overwhelming Splunk |
| **Decoupling** | AWS side and Splunk side don't need to know about each other's internals |
| **Dead Letter Queue (DLQ)** | Poison messages (malformed/unparsable) don't loop forever — they land in a DLQ for investigation |
| **Native Splunk Add-on support** | The Splunk Add-on for AWS has a purpose-built "SQS-Based S3" input type |

### 3.6 Splunk — What It's Doing Here

Splunk is a **log indexing, search, and visualization platform**. In this pipeline, Splunk (via the Splunk Add-on for AWS) is the **consumer**: it polls SQS, reads the message (which contains an S3 event notification describing the new CloudTrail log file), downloads that `.json.gz` file from the CloudTrail log bucket, decompresses it, parses each CloudTrail record as an individual event, extracts fields (`eventName`, `bucketName`, `sourceIPAddress`, etc.), and indexes them so they're instantly searchable with SPL.

---

## 4. Architecture

### 4.1 High-Level Flow

```
+-----------+   PutObject / GetObject / DeleteObject
|  User /   | ---------------------------------------+
|  Client   |                                         v
+-----------+                          +----------------------------+
                                        |  S3 Application Bucket     |
                                        |  (demo-upload-bucket)      |
                                        +--------------+---------------+
                                                       |
                                                       | API call observed by
                                                       v
                                        +----------------------------------+
                                        |  AWS CloudTrail                   |
                                        |  (Data Events enabled on the      |
                                        |   application bucket)             |
                                        +--------------+---------------------+
                                                       |
                                                       | writes compressed .json.gz log file
                                                       v
                                        +----------------------------------+
                                        |  CloudTrail Log Bucket            |
                                        |  (demo-cloudtrail-logs)           |
                                        +--------------+---------------------+
                                                       |
                                                       | S3 Event Notification
                                                       | (s3:ObjectCreated:*)
                                                       v
                                        +----------------------------------+
                                        |  Amazon SQS Queue                 |
                                        |  (cloudtrail-log-queue)           |
                                        |  + Dead Letter Queue (DLQ)        |
                                        +--------------+---------------------+
                                                       |
                                                       | polled by
                                                       v
                                        +----------------------------------+
                                        |  EC2 Instance                     |
                                        |  Splunk Enterprise +              |
                                        |  Splunk Add-on for AWS            |
                                        |  (SQS-Based S3 input)             |
                                        +--------------+---------------------+
                                                       |
                                                       | parses & indexes
                                                       v
                                        +----------------------------------+
                                        |  Splunk Index: cloudtrail_s3      |
                                        |  -> SPL Searches                  |
                                        |  -> Dashboards                    |
                                        |  -> Alerts                        |
                                        +----------------------------------+
```

**Plain-text version of the same flow (safe to paste anywhere, no formatting required):**

`Client → S3 Application Bucket → CloudTrail (Data Events) → CloudTrail Log Bucket → S3 Event Notification → SQS Queue (+ DLQ) → Splunk (EC2 + Add-on for AWS) → Splunk Index (cloudtrail_s3) → Dashboards / Alerts`

### 4.2 Why Not SNS in This Diagram?

As explained in Section 3.4, SNS is intentionally left out of the primary path. It is architecturally valid to insert SNS between the CloudTrail log bucket and SQS *only if* you need fan-out to multiple consumers. For this single-consumer demo, S3 → SQS directly is simpler, cheaper (one less hop), and easier to troubleshoot.

### 4.3 AWS Resource Naming Used in This Guide

| Resource | Name Used in This Guide | Replace With |
|---|---|---|
| Application (upload) bucket | `demo-upload-bucket-<yourname>` | globally unique name |
| CloudTrail log bucket | `demo-cloudtrail-logs-<yourname>` | globally unique name |
| CloudTrail trail | `demo-cloudtrail` | any name |
| SQS main queue | `cloudtrail-log-queue` | any name |
| SQS dead letter queue | `cloudtrail-log-queue-dlq` | any name |
| IAM user for Splunk | `splunk-aws-reader` | any name |
| EC2 instance | `splunk-server` | any name |
| AWS Region | `ap-south-1` (Mumbai) | your preferred region |

> S3 bucket names are globally unique across **all** AWS accounts. Append your name/initials or a random suffix, e.g. `demo-upload-bucket-saime01`.

---

## 5. Hands-On Lab Part 1 — S3 Buckets Foundation

### 5.1 Create the Application (Upload) Bucket

**Console steps:**

1. Open **AWS Console → S3 → Create bucket**.
2. **Bucket name:** `demo-upload-bucket-<yourname>`
3. **AWS Region:** `ap-south-1` (or your chosen region — must match everywhere in this guide).
4. **Block Public Access settings:** Leave **all four checkboxes ON** (Block all public access) — this bucket holds demo files and must not be public.
5. **Bucket Versioning:** Enable (optional for the app bucket, but good practice).
6. **Default encryption:** SSE-S3 (Amazon S3-managed keys) is fine for a demo.
7. Click **Create bucket**.

✅ **Verification checkpoint:** You should see `demo-upload-bucket-<yourname>` listed in your S3 console with region `ap-south-1`.

### 5.2 Create the CloudTrail Log Bucket

1. **S3 → Create bucket** again.
2. **Bucket name:** `demo-cloudtrail-logs-<yourname>`
3. Same region as above.
4. **Block Public Access:** all four ON.
5. **Bucket Versioning:** Enable — protects log integrity.
6. **Default encryption:** SSE-S3 is fine for demo (production should use SSE-KMS, see Section 14).
7. Click **Create bucket**.

✅ **Verification checkpoint:** Two buckets now exist — one for uploads, one for CloudTrail logs. **They must be different buckets.** Never point CloudTrail's log delivery back into the same bucket you're monitoring — this creates a feedback loop of logs describing logs.

### 5.3 Why Two Separate Buckets?

| Reason | Explanation |
|---|---|
| Avoid infinite log loops | If CloudTrail Data Events were enabled on the log bucket itself, every log delivery would generate a new `PutObject` event, which would generate another log, forever |
| Access control separation | The app bucket may need broader write access (users uploading); the log bucket should be locked down to CloudTrail's service principal and Splunk's read-only role only |
| Retention policy separation | Logs often need longer retention (compliance) than application data |

---

## 6. Hands-On Lab Part 2 — CloudTrail Configuration

### 6.1 Create the Trail

1. **AWS Console → CloudTrail → Trails → Create trail**.
2. **Trail name:** `demo-cloudtrail`
3. **Storage location:** Choose **"Use existing S3 bucket"** → select `demo-cloudtrail-logs-<yourname>`.
4. **Log file SSE-KMS encryption:** Leave disabled for the demo (enable in production — see Section 14).
5. **Log file validation:** Enable — this lets you cryptographically verify logs haven't been tampered with.
6. **SNS notification delivery:** Leave **disabled**. We are not using SNS in this design (Section 3.4).
7. **CloudWatch Logs:** Optional — leave disabled for this demo to keep it simple and free.
8. Click **Next**.

### 6.2 Configure Data Events (the critical step)

This is the step most tutorials skip or get wrong.

1. On the **"Choose log events"** page, scroll to **Data events**.
2. **Data event source:** Select **S3**.
3. Click **"Add S3 bucket"** (or "Browse").
4. Select `demo-upload-bucket-<yourname>` — **only this bucket, not "All current and future S3 buckets."**
5. Under **Logging type for selected S3 bucket**, check:
   - ✅ **Read** (captures `GetObject`, and other read-type S3 API calls)
   - ✅ **Write** (captures `PutObject`, `DeleteObject`, and other write-type S3 API calls)
6. Click **Next**, review the summary, then **Create trail**.

✅ **Verification checkpoint:** On the trail's detail page, under **Data events**, you should see one row: `demo-upload-bucket-<yourname>` with Read + Write both enabled.

> **Beginner explanation:** Checking "Write" means every time someone uploads or deletes a file in that bucket, CloudTrail writes a record. Checking "Read" does the same for downloads. For this demo, enable both so you can show all three event types (`PutObject`, `GetObject`, `DeleteObject`) in Splunk.

> **Enterprise Perspective:** In production, enabling "Read" on a high-traffic bucket can generate enormous log volume and cost. Many teams enable **Write only** on production buckets and add **Read** only during active incident investigation.

### 6.3 What Gets Written to the Log Bucket

After you perform an action on the app bucket, CloudTrail will (within minutes) write an object to the log bucket at a path like:

```
demo-cloudtrail-logs-<yourname>/
  AWSLogs/
    <account-id>/
      CloudTrail/
        ap-south-1/
          2026/
            07/
              03/
                <account-id>_CloudTrail_ap-south-1_20260703T1015Z_XXXXXXXXXXXXXXXX.json.gz
```

Each file is a **gzip-compressed JSON** containing an array of `Records`, one per API call.

---

## 7. Hands-On Lab Part 3 — Amazon SQS + Event Notification Wiring

### 7.1 Create the Dead Letter Queue First

1. **AWS Console → SQS → Create queue**.
2. **Type:** Standard.
3. **Name:** `cloudtrail-log-queue-dlq`
4. Leave defaults, click **Create queue**.

### 7.2 Create the Main Queue

1. **SQS → Create queue**.
2. **Type:** Standard.
3. **Name:** `cloudtrail-log-queue`
4. **Visibility timeout:** `300` seconds (5 minutes — gives Splunk enough time to download and process a log file before the message becomes visible again to another consumer).
5. **Message retention period:** `4 days` (default is fine for a demo).
6. **Dead-letter queue:** Enable → select `cloudtrail-log-queue-dlq` → **Maximum receives: 5** (after 5 failed processing attempts, the message moves to the DLQ instead of retrying forever).
7. Click **Create queue**.

### 7.3 Attach an Access Policy Allowing S3 to Send Messages

SQS queues are private by default — S3 is not allowed to publish to it until you explicitly grant permission.

1. Open the `cloudtrail-log-queue` → **Access policy** tab → **Edit**.
2. Replace/merge with the following policy (replace `<ACCOUNT_ID>` and bucket name):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ToSendMessage",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "SQS:SendMessage",
      "Resource": "arn:aws:sqs:ap-south-1:<ACCOUNT_ID>:cloudtrail-log-queue",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:s3:::demo-cloudtrail-logs-<yourname>"
        },
        "StringEquals": {
          "aws:SourceAccount": "<ACCOUNT_ID>"
        }
      }
    }
  ]
}
```

3. Click **Save**.

✅ **Verification checkpoint:** The policy explicitly scopes `SourceArn` to your CloudTrail log bucket only — this prevents any other S3 bucket (yours or another account's, if the queue were ever misconfigured) from spamming this queue.

### 7.4 Configure the S3 Event Notification on the CloudTrail Log Bucket

1. Go to **S3 → `demo-cloudtrail-logs-<yourname>` → Properties tab**.
2. Scroll to **Event notifications → Create event notification**.
3. **Event name:** `notify-splunk-sqs`
4. **Prefix (optional):** `AWSLogs/` (scopes the trigger to only the CloudTrail log path, good hygiene).
5. **Event types:** Check **"All object create events"** (`s3:ObjectCreated:*`).
6. **Destination:** Select **SQS Queue** → choose `cloudtrail-log-queue`.
7. Click **Save changes**.

✅ **Verification checkpoint:** Under the bucket's **Properties → Event notifications**, you should see `notify-splunk-sqs` listed, pointing to `cloudtrail-log-queue`.

### 7.5 Quick Manual Test (Before Involving Splunk)

Upload any small test file directly to the log bucket to prove the SQS wiring works end-to-end, independent of CloudTrail delivery timing:

```bash
echo "test" > test.txt
aws s3 cp test.txt s3://demo-cloudtrail-logs-<yourname>/AWSLogs/manual-test.txt
```

Then check the queue:

```bash
aws sqs receive-message \
  --queue-url https://sqs.ap-south-1.amazonaws.com/<ACCOUNT_ID>/cloudtrail-log-queue \
  --max-number-of-messages 1
```

✅ You should get back a JSON message body containing an `s3` event record referencing `manual-test.txt`. If you see this, the S3 → SQS wiring is confirmed working. **Delete this test message** (or just let it expire) — it's not a real CloudTrail log and would otherwise confuse Splunk later. If Splunk is not yet configured, you can safely leave it; the add-on will simply skip/error gracefully on a non-CloudTrail-format file, or you can purge the queue before starting Splunk ingestion:

```bash
aws sqs purge-queue --queue-url https://sqs.ap-south-1.amazonaws.com/<ACCOUNT_ID>/cloudtrail-log-queue
```

---

## 8. Hands-On Lab Part 4 — IAM for Splunk

### 8.1 Design Principle: Least Privilege

Splunk needs exactly three capabilities:
1. Receive and delete messages from the SQS queue.
2. Read objects from the CloudTrail log bucket.
3. List the CloudTrail log bucket (to resolve keys).

Nothing else. No write access, no access to the application bucket, no account-wide `s3:*`.

### 8.2 Create the IAM Policy

1. **IAM → Policies → Create policy → JSON tab**. Paste (replace `<ACCOUNT_ID>` and bucket/queue names):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadCloudTrailLogBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::demo-cloudtrail-logs-<yourname>",
        "arn:aws:s3:::demo-cloudtrail-logs-<yourname>/*"
      ]
    },
    {
      "Sid": "ConsumeSQSQueue",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes",
        "sqs:GetQueueUrl"
      ],
      "Resource": "arn:aws:sqs:ap-south-1:<ACCOUNT_ID>:cloudtrail-log-queue"
    }
  ]
}
```

2. **Name:** `SplunkS3CloudTrailReadPolicy` → **Create policy**.

### 8.3 Create the IAM User (Demo) or IAM Role (Recommended When Splunk Runs on EC2)

**Option A — IAM User with Access Keys (simplest for a first demo):**

1. **IAM → Users → Create user** → name: `splunk-aws-reader`.
2. **Do not** enable console access.
3. Attach policy: `SplunkS3CloudTrailReadPolicy`.
4. After creation, open the user → **Security credentials → Create access key** → choose **"Third-party service"** → download/copy the **Access Key ID** and **Secret Access Key**. You will paste these into the Splunk Add-on in Section 10.

**Option B — IAM Role attached to the EC2 instance (recommended, more secure, no long-lived keys):**

1. **IAM → Roles → Create role** → Trusted entity: **AWS service → EC2**.
2. Attach policy: `SplunkS3CloudTrailReadPolicy`.
3. **Role name:** `SplunkEC2Role`.
4. You will attach this role to the EC2 instance during launch in Section 9.

> **Enterprise Perspective:** Always prefer **Option B (IAM Role)** in real deployments. Long-lived access keys on disk are a leading cause of AWS credential leaks. This demo shows Option A because it's slightly faster to explain in an interview/demo setting, but you should build muscle memory for Option B.

---

## 9. Hands-On Lab Part 5 — EC2 + Splunk Enterprise Installation

### 9.1 Launch the EC2 Instance

| Setting | Value | Why |
|---|---|---|
| AMI | Ubuntu Server 22.04 LTS | Well-documented, stable, free-tier-friendly base |
| Instance type | `t3.medium` (2 vCPU, 4 GB RAM) | Splunk is memory-hungry; `t2.micro`/`t3.micro` will struggle or fail to start |
| Storage | 30 GB gp3 | Splunk + indexes need headroom beyond the OS |
| Key pair | Create/select one, download `.pem` | Needed for SSH |
| Network | Default VPC is fine for a demo | — |
| Public IP | Enabled | You'll access Splunk Web from your browser |
| IAM Role | Attach `SplunkEC2Role` if you chose Option B in 8.3 | Avoids hardcoding keys |

**Security Group — inbound rules:**

| Type | Port | Source |
|---|---|---|
| SSH | 22 | Your IP only |
| Custom TCP | 8000 | Your IP only (Splunk Web) |
| Custom TCP | 8089 | Your IP only (Splunk management API — optional, only if needed) |

> **Never** open 8000/8089 to `0.0.0.0/0` even for a demo — Splunk Web with a default/weak password exposed to the internet is a common real-world breach vector. Restrict to your IP.

Launch the instance, then connect:

```bash
chmod 400 my-key.pem
ssh -i my-key.pem ubuntu@<EC2_PUBLIC_IP>
```

### 9.2 Update the Server

```bash
sudo apt update && sudo apt upgrade -y
```

### 9.3 Download and Install Splunk Enterprise

1. On the EC2 instance, get the Splunk `.deb` package download link from [splunk.com/download/splunk-enterprise](https://www.splunk.com/en_us/download/splunk-enterprise.html) (requires a free Splunk.com account; copy the direct `wget` link shown after accepting the license).

```bash
# Example — replace with the current version link from the Splunk download page
wget -O splunk.deb "https://download.splunk.com/products/splunk/releases/9.x.x/linux/splunk-9.x.x-linux-amd64.deb"

sudo dpkg -i splunk.deb
# If dpkg reports missing dependencies, run:
sudo apt-get install -f -y
```

2. Splunk installs to `/opt/splunk`.

> **Note on running as root:** The steps below run Splunk as the `ubuntu`/root user for simplicity, which is fine for this demo. In production, install and run Splunk under a dedicated, non-root `splunk` service account (`sudo /opt/splunk/bin/splunk enable boot-start -user splunk`) per Splunk's hardening guidance.

### 9.4 Start Splunk and Accept the License

```bash
sudo /opt/splunk/bin/splunk start --accept-license
```

- You will be prompted to create an **admin username and password** on first start. Set a strong password — write it down.
- This also starts Splunk with a **60-day Enterprise trial license** by default, which is exactly what you want for this demo (up to 500 MB/day indexing).

### 9.5 Enable Splunk to Start on Boot

```bash
sudo /opt/splunk/bin/splunk enable boot-start
```

### 9.6 Verify the Service Is Running

```bash
sudo /opt/splunk/bin/splunk status
```

Expected output:

```
splunkd is running (PID: xxxx).
splunk helpers are running (PIDs: xxxx xxxx).
```

Also confirm the web port is listening:

```bash
sudo ss -tulpn | grep 8000
```

### 9.7 Log In to Splunk Web

Open a browser: `http://<EC2_PUBLIC_IP>:8000` → log in with the admin credentials set in Step 9.4.

✅ **Verification checkpoint:** You see the Splunk Enterprise home page ("Splunk Bar" with Search & Reporting app available).

---

## 10. Hands-On Lab Part 6 — Splunk Add-on for AWS Configuration

### 10.1 Install the Add-on

1. Inside Splunk Web: **Apps (top left) → Find More Apps**.
2. Search for **"Splunk Add-on for Amazon Web Services"**.
3. Click **Install**, log in with your Splunk.com credentials when prompted, accept the license.
4. Restart Splunk when prompted:

```bash
sudo /opt/splunk/bin/splunk restart
```

### 10.2 Verify the Add-on Is Installed

After the restart, go to **Apps** in the top navigation — you should see **"Splunk Add-on for AWS"** listed.

### 10.3 Add the AWS Account

1. Open **Splunk Add-on for AWS → Configuration → Account → Add**.
2. **Name:** `demo-aws-account`
3. **Key ID / Secret Key:** paste the values from Section 8.3 Option A (or select **"Use ARN"** / instance-role-based auth if you attached `SplunkEC2Role` in Option B — the add-on will detect and use the instance's IAM role automatically).
4. **Region category:** Global (or select `ap-south-1` region category as applicable).
5. Click **Add**.

✅ **Verification checkpoint:** The account appears in the account list with a green/valid status, not an error icon.

### 10.4 Create the SQS-Based S3 Input

This is the core ingestion input — it tells Splunk: *"Poll this SQS queue; whenever a message arrives, go download the referenced S3 object and index it."*

1. **Splunk Add-on for AWS → Inputs → Create New Input → SQS-Based S3**.
2. **Name:** `cloudtrail-sqs-input`
3. **AWS Account:** `demo-aws-account`
4. **AWS Region:** `ap-south-1`
5. **SQS Queue Name:** select `cloudtrail-log-queue` from the dropdown.
6. **SQS Batch Size:** `10` (default is fine for a demo).
7. **Source type:** select **`aws:cloudtrail`** (built-in sourcetype that the add-on ships with — it already knows how to parse CloudTrail JSON fields).
8. **Index:** click **Create a new index** → name it `cloudtrail_s3` → **Save** → then select it here.
9. Leave **"Also poll data from S3 bucket associated with the SQS queue"** = enabled (this is what actually makes it download and parse the referenced object, not just index the SQS notification text).
10. Click **Add** / **Save**.

✅ **Verification checkpoint:** The new input `cloudtrail-sqs-input` appears in the Inputs list with status **Enabled**.

> **Beginner explanation of what happens next:** Every ~few seconds, this input polls the queue. When it finds a message (which is really just "hey, a new file landed at this S3 path"), it fetches that file from `demo-cloudtrail-logs-<yourname>`, unzips the `.json.gz`, and indexes each `Records[]` entry inside it as one Splunk event, tagged with sourcetype `aws:cloudtrail` in index `cloudtrail_s3`. It then deletes the SQS message so it isn't reprocessed.

---

## 11. End-to-End Testing & Verification

### 11.1 Generate Real Traffic

From your local machine or CloudShell (with credentials that have access to the *application* bucket, not necessarily the Splunk reader user):

```bash
# Upload
echo "This is a demo resume" > resume.pdf
aws s3 cp resume.pdf s3://demo-upload-bucket-<yourname>/resume.pdf

# Download
aws s3 cp s3://demo-upload-bucket-<yourname>/resume.pdf ./downloaded-resume.pdf

# Delete
aws s3 rm s3://demo-upload-bucket-<yourname>/resume.pdf
```

### 11.2 Timeline of What Happens (For Your Demo Narration)

| Time | What's Happening |
|---|---|
| T+0 min | You run `aws s3 cp` — the `PutObject` API call happens |
| T+2–15 min | CloudTrail batches and delivers the Data Event as a `.json.gz` file to the log bucket (CloudTrail delivery is near-real-time but not instant — typically under 15 minutes, occasionally up to ~30) |
| T+seconds after delivery | S3 Event Notification fires → message lands in `cloudtrail-log-queue` |
| T+seconds after that | Splunk's SQS-Based S3 input polls, finds the message, downloads and parses the log file |
| Result | Event is now searchable in Splunk index `cloudtrail_s3` |

> This delay is the single most common point of confusion for people new to this pipeline: **"I uploaded a file and nothing shows up in Splunk!"** — the answer is almost always *"wait a few more minutes; CloudTrail hasn't delivered the log file yet."* See Section 13 for how to distinguish this from an actual misconfiguration.

### 11.3 Verify in Splunk

1. Go to **Search & Reporting** app in Splunk Web.
2. Set the time range picker to **"Last 60 minutes."**
3. Run:

```spl
index=cloudtrail_s3 sourcetype=aws:cloudtrail
```

✅ **Verification checkpoint:** You should see events appear, each with fields like `eventName`, `bucketName` (inside `requestParameters`), `sourceIPAddress`, `userIdentity.arn`, `eventTime`.

If nothing appears after 20–30 minutes, go directly to Section 13 (Troubleshooting).

---

## 12. Splunk SPL Searches & Dashboard Build

### 12.1 Foundational SPL Queries

**All CloudTrail events in this pipeline:**

```spl
index=cloudtrail_s3 sourcetype=aws:cloudtrail
```

**Uploads only:**

```spl
index=cloudtrail_s3 sourcetype=aws:cloudtrail eventName=PutObject
```

**Downloads only:**

```spl
index=cloudtrail_s3 sourcetype=aws:cloudtrail eventName=GetObject
```

**Deletes only:**

```spl
index=cloudtrail_s3 sourcetype=aws:cloudtrail eventName=DeleteObject
```

**Filter to one bucket:**

```spl
index=cloudtrail_s3 sourcetype=aws:cloudtrail requestParameters.bucketName="demo-upload-bucket-<yourname>"
```

**Filter to one IAM identity:**

```spl
index=cloudtrail_s3 sourcetype=aws:cloudtrail userIdentity.arn="arn:aws:iam::<ACCOUNT_ID>:user/admin"
```

**Failed API calls (permission errors, etc.):**

```spl
index=cloudtrail_s3 sourcetype=aws:cloudtrail errorCode=*
```

**Top uploading IAM users (last 24h):**

```spl
index=cloudtrail_s3 sourcetype=aws:cloudtrail eventName=PutObject
| stats count by userIdentity.arn
| sort -count
```

**Upload activity over time (time chart):**

```spl
index=cloudtrail_s3 sourcetype=aws:cloudtrail eventName=PutObject
| timechart span=5m count
```

**Top source IP addresses:**

```spl
index=cloudtrail_s3 sourcetype=aws:cloudtrail
| stats count by sourceIPAddress
| sort -count
| head 10
```

### 12.2 Build the Dashboard

1. Run any of the searches above → click **Save As → Dashboard Panel**.
2. Create a new dashboard: **S3 CloudTrail Activity Monitor**.
3. Repeat for each panel, choosing an appropriate visualization:

| Panel | SPL Base | Visualization |
|---|---|---|
| Total Uploads | `eventName=PutObject \| stats count` | Single Value |
| Total Downloads | `eventName=GetObject \| stats count` | Single Value |
| Total Deletes | `eventName=DeleteObject \| stats count` | Single Value |
| Upload Trend | `eventName=PutObject \| timechart span=5m count` | Line Chart |
| Top IAM Users | `stats count by userIdentity.arn \| sort -count` | Bar Chart |
| Top Source IPs | `stats count by sourceIPAddress \| sort -count \| head 10` | Bar Chart |
| Most Accessed Buckets | `stats count by requestParameters.bucketName` | Pie Chart |
| Failed API Calls | `errorCode=* \| table _time, eventName, errorCode, userIdentity.arn` | Table |
| Recent Activity | `\| table _time, eventName, requestParameters.bucketName, requestParameters.key, sourceIPAddress` | Table |

4. Arrange panels, **Save**.

✅ **Verification checkpoint:** Your dashboard shows live numbers matching the uploads/downloads/deletes you performed in Section 11.1.

---

## 13. Alerts, Troubleshooting & Failure Scenarios

### 13.1 Create a Basic Security Alert

**Use case:** Alert when a `DeleteObject` happens outside expected automation.

1. Run this search:

```spl
index=cloudtrail_s3 sourcetype=aws:cloudtrail eventName=DeleteObject
```

2. **Save As → Alert**.
3. **Trigger condition:** `Number of Results > 0`
4. **Time range:** Run every 5 minutes over the last 5 minutes.
5. **Trigger action:** Send email (or add a webhook action if integrating with Slack/PagerDuty).
6. Save.

### 13.2 Common Failure Scenarios & Fixes

| Symptom | Likely Cause | Fix |
|---|---|---|
| Nothing in Splunk after 30+ minutes | CloudTrail hasn't delivered yet | Wait — CloudTrail delivery can take up to 15–30 min. Check the CloudTrail log bucket directly in S3 console for new `.json.gz` files first |
| Files exist in the log bucket, but nothing in Splunk | SQS notification not firing | Re-check Section 7.4 — event type must be "All object create events" and destination must be the correct queue |
| Files in log bucket, SQS has messages, but Splunk shows nothing | IAM permissions issue | Check `/opt/splunk/var/log/splunk/splunk_ta_aws_*.log` for `AccessDenied` errors; re-verify the IAM policy from Section 8.2 |
| Splunk Add-on shows account status "Invalid" | Wrong Access Key/Secret, or region mismatch | Re-enter credentials; confirm the IAM user's keys weren't rotated/deleted |
| SQS queue depth growing, never draining | Splunk input disabled, or Splunk service down | `sudo /opt/splunk/bin/splunk status`; check input is Enabled in Add-on Inputs page |
| Duplicate events in Splunk | Visibility timeout too short, message reprocessed before delete completed | Increase SQS visibility timeout (Section 7.2) to comfortably exceed processing time |
| Messages piling up in the DLQ | Malformed messages (e.g., your manual test file from 7.5) or the add-on failing to parse a file | Inspect DLQ messages (`aws sqs receive-message` against the DLQ URL); purge stale test messages |
| `PermissionDenied` in Splunk log when downloading S3 object | IAM policy resource ARN typo, or bucket name mismatch | Double check exact bucket name/ARN in the policy from Section 8.2 |
| Splunk Web won't load | Security group blocking port 8000, or Splunk not started | `sudo ss -tulpn \| grep 8000`; confirm your IP is allowed in the security group |
| High RAM usage / instance freezing | Undersized instance | Confirm you used `t3.medium` or larger, not `t2.micro`/`t3.micro` |

### 13.3 How to Read the Add-on's Internal Logs

```bash
sudo tail -f /opt/splunk/var/log/splunk/splunk_ta_aws_sqs_based_s3.log
```

This shows exactly what the input is doing each polling cycle — queue polls, messages received, S3 downloads attempted, and any errors with full stack traces.

### 13.4 How to Manually Inspect an SQS Message (Bypass Splunk Entirely)

If you're unsure whether the problem is on the AWS side or the Splunk side, isolate it:

```bash
aws sqs receive-message \
  --queue-url https://sqs.ap-south-1.amazonaws.com/<ACCOUNT_ID>/cloudtrail-log-queue \
  --max-number-of-messages 1 \
  --visibility-timeout 5
```

- If you **get a message back** → the AWS side (S3 → CloudTrail → Log Bucket → SQS) is working; the problem is on the Splunk side (IAM creds, input config, Splunk service health).
- If you **get nothing back** and you're sure a file landed in the log bucket → the problem is upstream (event notification misconfigured, or SQS access policy blocking S3, per Section 7.3–7.4).

---

## 14. Best Practices, Cost, Cleanup & Interview Cheat Sheet

### 14.1 Production Hardening Checklist (Beyond This Demo)

- [ ] Enable **SSE-KMS** (not just SSE-S3) on the CloudTrail log bucket, with a dedicated KMS key and a key policy granting Splunk's role `kms:Decrypt` only.
- [ ] Enable **S3 Object Lock (Compliance mode)** or bucket policy denying `DeleteObject`/`PutBucketPolicy` on the log bucket to make logs tamper-resistant.
- [ ] Scope CloudTrail Data Events to specific buckets/prefixes, not "all current and future" — control cost.
- [ ] Use an **IAM Role** on the EC2 instance (or migrate to Splunk Cloud with proper federated auth) instead of static access keys.
- [ ] Put the Splunk EC2 instance in a private subnet behind a bastion/VPN — do not expose Splunk Web (`8000`) to the public internet, ever, even with a security group restriction, in a real production environment.
- [ ] Enable CloudTrail **log file integrity validation** and periodically verify with `aws cloudtrail validate-logs`.
- [ ] Configure **S3 Lifecycle policies** on the log bucket (e.g., transition to Glacier after 90 days, per your compliance retention requirement).
- [ ] Set CloudWatch alarms on SQS `ApproximateAgeOfOldestMessage` to detect a stalled pipeline before Splunk consumers notice.
- [ ] Monitor the DLQ — an empty DLQ over time is healthy; a growing DLQ indicates systemic parsing failures.
- [ ] Consider **Kinesis Data Firehose** instead of raw SQS polling if event volume grows into the millions/day range, for better throughput and native Splunk HEC integration.

### 14.2 Cleanup Steps (Run When You're Done With the Demo)

Run in this order to avoid dependency errors:

```bash
# 1. Delete SQS event notification wiring (console: remove the event notification from the log bucket)

# 2. Delete SQS queues
aws sqs delete-queue --queue-url https://sqs.ap-south-1.amazonaws.com/<ACCOUNT_ID>/cloudtrail-log-queue
aws sqs delete-queue --queue-url https://sqs.ap-south-1.amazonaws.com/<ACCOUNT_ID>/cloudtrail-log-queue-dlq

# 3. Stop/delete the CloudTrail trail
aws cloudtrail stop-logging --name demo-cloudtrail
aws cloudtrail delete-trail --name demo-cloudtrail

# 4. Empty and delete both S3 buckets
aws s3 rm s3://demo-upload-bucket-<yourname> --recursive
aws s3api delete-bucket --bucket demo-upload-bucket-<yourname>

aws s3 rm s3://demo-cloudtrail-logs-<yourname> --recursive
aws s3api delete-bucket --bucket demo-cloudtrail-logs-<yourname>

# 5. Terminate the EC2 instance
aws ec2 terminate-instances --instance-ids <INSTANCE_ID>

# 6. Delete the IAM user/role and policy
aws iam detach-user-policy --user-name splunk-aws-reader --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/SplunkS3CloudTrailReadPolicy
aws iam delete-access-key --user-name splunk-aws-reader --access-key-id <ACCESS_KEY_ID>
aws iam delete-user --user-name splunk-aws-reader
aws iam delete-policy --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/SplunkS3CloudTrailReadPolicy
```

> If you attached versioning to the buckets, you must delete all object versions and delete markers before the bucket delete will succeed — use `aws s3api list-object-versions` + `delete-objects`, or delete via console which handles this for you with a confirmation step.

### 14.3 Interview Q&A Cheat Sheet

**Q: Does S3 log object uploads by default?**
A: No. S3 offers optional Server Access Logging (delayed, flat-file), but the structured, near-real-time, IAM-context-rich method is enabling **CloudTrail Data Events** on the bucket.

**Q: What's the difference between CloudTrail Management Events and Data Events?**
A: Management Events log control-plane API calls (creating/deleting resources) and are enabled by default with 90-day free history. Data Events log data-plane operations (like `PutObject`/`GetObject` on S3, or `Invoke` on Lambda), are **not** enabled by default, and are billed per event.

**Q: Why can't you send S3 event notifications directly to Splunk via SNS?**
A: Splunk is not a supported native SNS subscriber type (SNS supports email, SMS, SQS, Lambda, HTTP/HTTPS, mobile push). The Splunk Add-on for AWS is built to poll SQS, so the standard integration is S3 → SQS (directly, or via SNS fan-out to SQS if multiple consumers are needed) → Splunk.

**Q: Why put the event notification on the CloudTrail log bucket instead of the application bucket?**
A: Because what triggers Splunk ingestion is the arrival of a new CloudTrail *log file*, not the user's original upload. The application bucket is being watched by CloudTrail (Data Events); the log bucket is what produces the S3-native `ObjectCreated` event that we actually wire to SQS.

**Q: Why use two separate S3 buckets?**
A: To avoid a feedback loop (log bucket describing its own log deliveries) and to apply different access control/retention policies to application data vs. audit logs.

**Q: What's the purpose of the SQS Dead Letter Queue here?**
A: To capture messages that fail processing repeatedly (e.g., malformed files) so they don't loop forever and can be investigated separately, keeping the main queue healthy.

**Q: How would you scale this design for a high-traffic production bucket?**
A: Scope Data Events narrowly (specific prefixes/buckets, Write-only where possible), consider Kinesis Data Firehose instead of SQS polling for higher throughput and native Splunk HEC delivery, encrypt logs with SSE-KMS, and enforce log immutability with S3 Object Lock.

**Q: How long does it take for an S3 action to show up in Splunk in this design?**
A: Typically a few minutes to under 15–30 minutes — dominated by CloudTrail's own log delivery latency, not the SQS/Splunk polling steps (which are near-instant, on the order of seconds).

---

## Appendix A — Full End-to-End Summary (One-Paragraph Version for a Live Demo)

*"When a user uploads a file to my S3 application bucket, AWS CloudTrail — which I've configured with Data Events on that specific bucket — records the `PutObject` API call, including the IAM identity, source IP, and timestamp. CloudTrail writes that record as a compressed JSON log file into a separate, dedicated CloudTrail log bucket. That bucket has an S3 Event Notification configured to publish a message directly to an Amazon SQS queue whenever a new log file lands — I use SQS instead of SNS because Splunk doesn't support SNS as a native subscriber. On the Splunk side, I'm running Splunk Enterprise on an EC2 instance with the Splunk Add-on for AWS installed, configured with an SQS-Based S3 input that polls that queue, downloads and parses each CloudTrail log file, and indexes every event as a searchable record. From there I built SPL searches and a dashboard showing uploads, downloads, deletes, top users, and top source IPs — plus an alert that fires on any `DeleteObject` event."*

---

## Appendix B — Quick Reference: All Resource Names Used

```
S3 Application Bucket   : demo-upload-bucket-<yourname>
S3 CloudTrail Log Bucket: demo-cloudtrail-logs-<yourname>
CloudTrail Trail        : demo-cloudtrail
SQS Main Queue          : cloudtrail-log-queue
SQS Dead Letter Queue   : cloudtrail-log-queue-dlq
IAM Policy              : SplunkS3CloudTrailReadPolicy
IAM User (Option A)     : splunk-aws-reader
IAM Role (Option B)     : SplunkEC2Role
EC2 Instance            : splunk-server (t3.medium, Ubuntu 22.04 LTS)
Splunk Index            : cloudtrail_s3
Splunk Sourcetype       : aws:cloudtrail
Splunk SQS Input Name   : cloudtrail-sqs-input
Region                  : ap-south-1
```

---

*End of guide. This document is self-contained — no external references are required to complete the lab.*
