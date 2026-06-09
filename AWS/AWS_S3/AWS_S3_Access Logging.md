
# End-to-End Practical: Configure Amazon S3 Server Access Logging (Without Athena)

# Objective

Configure **Amazon S3 Server Access Logging** so that every request made to a source bucket is automatically recorded and stored in a separate logging bucket.

---

# Architecture

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/7a407d4e-224c-4947-b5cb-ef72a4f76b6d" />

---

# What are we going to create?

| Resource       | Name                 |
| -------------- | -------------------- |
| Source Bucket  | app-data-bucket-demo |
| Logging Bucket | app-access-logs-demo |

---

# Prerequisites

* AWS Account
* IAM User with S3 permissions
* AWS Management Console access

---

# Step 1: Login to AWS Console

1. Open AWS Console.
2. Search for **S3**.
3. Open the S3 service.

You should now see the Buckets page.

---

# Step 2: Create the Source Bucket

Click **Create bucket**.

## General Configuration

Bucket Name:

```
app-data-bucket-demo
```

Region:

```
Asia Pacific (Mumbai)
```

(or your preferred Region)

Object Ownership:

```
ACLs disabled (Recommended)
```

Block Public Access:

```
Keep all enabled
```

Bucket Versioning:

```
Disabled
```

Encryption:

```
SSE-S3 (Default)
```

Click **Create Bucket**.

---

# Step 3: Create the Logging Bucket

Click **Create bucket** again.

Bucket Name:

```
app-access-logs-demo
```

Keep all remaining settings as default.

Click **Create Bucket**.

---

# Step 4: Verify Both Buckets

You should now have:

```
app-data-bucket-demo

app-access-logs-demo
```

---

# Step 5: Open the Source Bucket

Open:

```
app-data-bucket-demo
```

Go to

```
Properties
```

Scroll down.

Find

```
Server Access Logging
```

Click

```
Edit
```

---

# Step 6: Enable Server Access Logging

Select

```
Enable
```

Destination Bucket:

```
app-access-logs-demo
```

Log Prefix (Optional):

```
logs/
```

Click

```
Save Changes
```

Server Access Logging is now enabled.

---

# Step 7: Upload Test Files

Open

```
app-data-bucket-demo
```

Click

```
Upload
```

Upload:

```
resume.pdf

photo.png

test.txt
```

Click

```
Upload
```

Each upload generates an access request that will later appear in the log bucket.

---

# Step 8: Download a File

Select

```
resume.pdf
```

Click

```
Download
```

This creates a **REST.GET.OBJECT** log entry.

---

# Step 9: View File Properties

Click

```
photo.png
```

Open its details.

AWS records another access request.

---

# Step 10: Delete a File

Select

```
test.txt
```

Click

```
Delete
```

Type

```
permanently delete
```

Confirm.

AWS records a delete request.

---

# Step 11: List Objects Using AWS CLI (Optional)

```bash
aws s3 ls s3://app-data-bucket-demo
```

This also creates an access log entry.

---

# Step 12: Upload a File Using AWS CLI

```bash
echo "Hello AWS" > demo.txt

aws s3 cp demo.txt s3://app-data-bucket-demo
```

---

# Step 13: Download Using AWS CLI

```bash
aws s3 cp s3://app-data-bucket-demo/demo.txt .
```

---

# Step 14: Delete Using AWS CLI

```bash
aws s3 rm s3://app-data-bucket-demo/demo.txt
```

---

# Step 15: Wait for Log Delivery

Server Access Logs are **not delivered instantly**.

Wait approximately:

```
10–30 minutes
```

Sometimes delivery can take longer depending on activity.

---

# Step 16: Open the Logging Bucket

Open

```
app-access-logs-demo
```

Open

```
logs/
```

You should see files similar to:

```
logs/

20260609-123000-ABCD123.log

20260609-124500-EFGH456.log

20260609-130000-HIJK789.log
```

These files are generated automatically by AWS.

---

# Step 17: Open a Log File

Click a log object.

Download it.

Open it with a text editor.

Example:

```
79a59df900b949e55d96a1e698f0

app-data-bucket-demo

[09/Jun/2026:10:15:23 +0000]

192.168.1.10

arn:aws:iam::123456789012:user/admin

REST.GET.OBJECT

resume.pdf

"GET /resume.pdf HTTP/1.1"

200

1024

2048

15

10

"-"

"AWS-CLI/2.15"
```

---

# Step 18: Understand the Log

| Field      | Meaning              |
| ---------- | -------------------- |
| Bucket     | app-data-bucket-demo |
| Operation  | REST.GET.OBJECT      |
| Object     | resume.pdf           |
| Requester  | IAM User             |
| IP Address | Client IP            |
| Status     | 200 = Success        |
| User Agent | Browser or AWS CLI   |

---

# Step 19: Test More Operations

Perform additional actions:

* Upload a file
* Download a file
* Delete a file
* List bucket contents
* Copy an object
* Rename an object (copy + delete)

Wait again.

Open the latest log file.

You will find corresponding entries for each action.

---

# Step 20: Enable Lifecycle Rule (Recommended)

Open

```
app-access-logs-demo

Management

Lifecycle Rules

Create Rule
```

Rule Name:

```
DeleteOldLogs
```

Choose:

```
Expire current versions
```

Days:

```
365
```

Save.

This automatically removes old log files.

---

# Step 21: Verify Logging is Working

Repeat:

* Upload a file
* Download it
* Delete it

Wait for log delivery.

Open the newest log file.

Verify that the latest operations appear.

If they do, the implementation is successful.

---

# Best Practices

* Use a **separate bucket** for logs.
* Never use the same bucket as both source and destination.
* Keep Block Public Access enabled.
* Apply lifecycle rules to reduce storage costs.
* Restrict read access to log files.
* Organize logs with a prefix such as `logs/`.
* Monitor storage growth over time.

---

# Final End-to-End Workflow

```text
                User / Application
                        │
        Upload / Download / Delete
                        │
                        ▼
          +---------------------------+
          | Source S3 Bucket          |
          | app-data-bucket-demo      |
          +---------------------------+
                        │
         Server Access Logging Enabled
                        │
                        ▼
             AWS S3 Logging Service
                        │
                        ▼
          +---------------------------+
          | Logging S3 Bucket         |
          | app-access-logs-demo      |
          +---------------------------+
                        │
                Log Files Created
                        │
                        ▼
             Download & Review Logs
             for Audit and Security
```

# Lab Validation Checklist

✅ Source bucket created

✅ Logging bucket created

✅ Server Access Logging enabled

✅ Test objects uploaded

✅ Objects downloaded

✅ Objects deleted

✅ Log files generated

✅ Log files verified

✅ Lifecycle rule configured

✅ End-to-end implementation completed successfully
