# AWS Hands-on Lab (Production Grade)

# Ensure At-Rest Encryption is Enabled When AWS Glue Writes Data to Amazon S3

## Lab Objective

In this hands-on lab, you will configure **AWS Glue** so that every object written to **Amazon S3** is encrypted **at rest** using **AWS KMS (SSE-KMS)**.

This guide follows the exact sequence used in production environments and includes:

* What
* Why
* Architecture
* Prerequisites
* Console clicks
* AWS CLI commands
* IAM permissions
* Glue Security Configuration
* Verification
* Troubleshooting
* Interview Questions

No steps are skipped.

---

# What is Encryption at Rest?

Encryption at rest means that data stored on disk is encrypted.

Even if someone gets direct access to the storage media, they cannot read the data without the encryption key.

Example:

Without encryption

```
employees.csv

Name

Salary

Password
```

Anyone with storage access can read it.

With encryption

```
QWERTYUIOP1234567890ASDFGHJKLZXCVBNM
```

Only AWS KMS can decrypt it.

---

# Why is Encryption Required?

Many compliance standards require encryption at rest.

* PCI-DSS
* HIPAA
* GDPR
* ISO 27001
* SOC2

Most enterprise organizations require all S3 objects to be encrypted.

---

# Real World Scenario

```
Application

↓

CSV File

↓

Amazon S3 Source Bucket

↓

AWS Glue ETL Job

↓

Amazon S3 Destination Bucket

↓

Athena

↓

QuickSight
```

Security team requirement:

> Every file written by AWS Glue must be encrypted using AWS KMS.

---

# Architecture

```
                    employees.csv
                           │
                           ▼
               Amazon S3 Source Bucket
                           │
                           ▼
                    AWS Glue Crawler
                           │
                           ▼
                     Glue Data Catalog
                           │
                           ▼
                      AWS Glue ETL Job
                           │
             Glue Security Configuration
                     (SSE-KMS Enabled)
                           │
                           ▼
             Amazon S3 Destination Bucket
                           │
                           ▼
                  AWS KMS Customer Key
```

---

# Prerequisites

You need

* AWS Account
* IAM User with AdministratorAccess (Lab only)
* AWS Glue
* Amazon S3
* AWS KMS
* IAM
* AWS CLI configured

Verify CLI

```bash
aws configure

aws sts get-caller-identity
```

---

# Step 1 - Create Source S3 Bucket

AWS Console

```
AWS Console

↓

Search

↓

Amazon S3

↓

Create Bucket
```

Bucket Name

```
glue-source-demo-12345
```

Region

```
ap-south-1
```

Block Public Access

```
Enabled
```

Leave remaining settings as default.

Click

```
Create Bucket
```

---

# Step 2 - Upload Sample File

Open bucket

```
glue-source-demo-12345

↓

Upload

↓

Add Files
```

Create local file

```
employees.csv
```

Contents

```csv
id,name,salary

1,Alice,50000

2,Bob,70000

3,John,90000
```

Click

```
Upload
```

Verify file exists.

---

# Step 3 - Create Destination Bucket

```
Amazon S3

↓

Create Bucket
```

Bucket Name

```
glue-output-demo-12345
```

Leave all settings as default.

Click

```
Create Bucket
```

Do **not** enable bucket encryption yet.

First configure Glue encryption.

---

# Step 4 - Create Customer Managed KMS Key

AWS Console

```
Search

↓

KMS

↓

Customer Managed Keys

↓

Create Key
```

Key Type

```
Symmetric
```

Key Usage

```
Encrypt and Decrypt
```

Click

```
Next
```

Alias

```
alias/glue-demo-key
```

Description

```
Glue Output Encryption Key
```

Click

```
Next

↓

Next

↓

Finish
```

Copy

```
Key ARN
```

Example

```
arn:aws:kms:ap-south-1:111111111111:key/xxxxxxxx
```

Save it.

---

# Step 5 - Create IAM Role for Glue

```
IAM

↓

Roles

↓

Create Role
```

Trusted Entity

```
AWS Service
```

Choose

```
Glue
```

Click

```
Next
```

Attach Policy

```
AWSGlueServiceRole
```

Click

```
Next
```

Role Name

```
GlueEncryptionRole
```

Create Role.

---

# Step 6 - Add S3 Permissions

Open

```
GlueEncryptionRole

↓

Add Permissions

↓

Create Inline Policy
```

Paste

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::glue-source-demo-12345/*",
        "arn:aws:s3:::glue-output-demo-12345/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::glue-source-demo-12345",
        "arn:aws:s3:::glue-output-demo-12345"
      ]
    }
  ]
}
```

Save.

---

# Step 7 - Add KMS Permissions

Again

```
GlueEncryptionRole

↓

Create Inline Policy
```

Paste

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect":"Allow",
      "Action":[
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:GenerateDataKey",
        "kms:DescribeKey",
        "kms:ReEncrypt*"
      ],
      "Resource":"YOUR_KMS_KEY_ARN"
    }
  ]
}
```

Replace

```
YOUR_KMS_KEY_ARN
```

with your actual KMS ARN.

Save.

---

# Step 8 - Create Glue Security Configuration

```
AWS Glue

↓

Administration

↓

Security Configurations

↓

Create Security Configuration
```

Name

```
Glue-KMS-Encryption
```

---

## CloudWatch Logs

Encryption

```
SSE-KMS
```

Key

```
alias/glue-demo-key
```

---

## Job Bookmark

Encryption

```
SSE-KMS
```

Key

```
alias/glue-demo-key
```

---

## Amazon S3

Encryption

```
SSE-KMS
```

Key

```
alias/glue-demo-key
```

Click

```
Create
```

This is the most important step in the lab.

---

# Step 9 - Create Glue Database

```
Glue

↓

Databases

↓

Add Database
```

Database Name

```
demo-db
```

Create.

---

# Step 10 - Create Glue Crawler

```
Glue

↓

Crawlers

↓

Create Crawler
```

Crawler Name

```
employee-crawler
```

Data Source

```
S3
```

Path

```
s3://glue-source-demo-12345/
```

IAM Role

```
GlueEncryptionRole
```

Database

```
demo-db
```

Finish wizard.

Click

```
Run
```

Wait until

```
Completed
```

Verify table

```
employees
```

exists.

---

# Step 11 - Create Glue ETL Job

```
Glue

↓

ETL Jobs

↓

Visual ETL

↓

Create
```

Source

```
Glue Catalog

↓

employees
```

Target

```
Amazon S3
```

Output Path

```
s3://glue-output-demo-12345/output/
```

IAM Role

```
GlueEncryptionRole
```

---

# Step 12 - Attach Security Configuration

Job Details

```
Advanced Properties

↓

Security Configuration

↓

Glue-KMS-Encryption
```

If you skip this step, Glue may write objects without using your KMS configuration.

---

# Step 13 - Save and Run Job

Click

```
Save

↓

Run
```

Wait

```
Succeeded
```

---

# Step 14 - Verify Output

```
Amazon S3

↓

glue-output-demo-12345

↓

output
```

Open generated file.

Click

```
Properties
```

Verify

```
Server-side Encryption

AWS KMS
```

KMS Key

```
alias/glue-demo-key
```

Success.

---

# Step 15 - Verify Using AWS CLI

List files

```bash
aws s3 ls s3://glue-output-demo-12345/output/
```

Copy object key.

Run

```bash
aws s3api head-object \
--bucket glue-output-demo-12345 \
--key output/part-00000.parquet
```

Expected

```json
{
  "ServerSideEncryption": "aws:kms",
  "SSEKMSKeyId": "arn:aws:kms:ap-south-1:xxxxxxxx"
}
```

This confirms encryption at rest.

---

# Step 16 - Enable Default Bucket Encryption (Best Practice)

```
Amazon S3

↓

glue-output-demo-12345

↓

Properties

↓

Default Encryption

↓

Edit
```

Choose

```
AWS KMS Key
```

Select

```
alias/glue-demo-key
```

Save.

Now:

* Glue encrypts objects.
* S3 also enforces encryption.

This is enterprise best practice.

---

# Step 17 - Test Failure Scenario

Remove this permission:

```json
kms:GenerateDataKey
```

Run Glue Job again.

Expected error:

```
AccessDeniedException

User is not authorized to perform

kms:GenerateDataKey
```

Restore permission.

Run again.

Job succeeds.

---

# Step 18 - Verify KMS Usage

```
AWS KMS

↓

Customer Managed Keys

↓

alias/glue-demo-key

↓

Monitoring
```

Observe API usage:

* Encrypt
* Decrypt
* GenerateDataKey

The counters increase after the Glue job runs.

---

# AWS CLI Commands

Create bucket

```bash
aws s3 mb s3://glue-source-demo-12345

aws s3 mb s3://glue-output-demo-12345
```

Upload file

```bash
aws s3 cp employees.csv s3://glue-source-demo-12345/
```

List objects

```bash
aws s3 ls s3://glue-source-demo-12345/

aws s3 ls s3://glue-output-demo-12345/
```

Verify encryption

```bash
aws s3api head-object \
--bucket glue-output-demo-12345 \
--key output/part-00000.parquet
```

Verify bucket encryption

```bash
aws s3api get-bucket-encryption \
--bucket glue-output-demo-12345
```

---

# Troubleshooting

## Glue Job Failed

Check:

* IAM Role attached
* Security Configuration attached
* KMS permissions
* S3 permissions

---

## AccessDeniedException

Verify:

```
kms:Encrypt

kms:Decrypt

kms:GenerateDataKey

kms:DescribeKey
```

exist in the Glue IAM Role and KMS key policy.

---

## Output Not Encrypted

Verify:

* Security Configuration attached to the Glue Job
* S3 output encryption set to SSE-KMS
* KMS key exists and is enabled

---

# Interview Questions

## How do you ensure AWS Glue writes encrypted objects to S3?

Attach a **Glue Security Configuration** with **SSE-KMS** enabled and grant the Glue IAM role permissions to use the KMS key.

---

## Is enabling S3 bucket encryption alone enough?

No. Bucket default encryption protects objects written to the bucket, but using **Glue Security Configuration + KMS + bucket default encryption** provides layered security and stronger governance.

---

## How do you verify encryption?

Run:

```bash
aws s3api head-object \
--bucket glue-output-demo-12345 \
--key output/part-00000.parquet
```

Look for:

```
ServerSideEncryption

aws:kms
```

---

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/19c90d07-a3a9-4a50-a038-543a9a6606bf" />


# Final Learning Outcome

After completing this lab, you will be able to:

* Configure AWS KMS for Glue
* Create and use Glue Security Configurations
* Encrypt Glue output using SSE-KMS
* Configure IAM permissions correctly
* Verify encryption from both the AWS Console and CLI
* Troubleshoot KMS-related Glue failures
* Explain this complete workflow confidently in AWS, DevOps, and Data Engineering interviews.

