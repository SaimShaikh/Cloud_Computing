# AWS Hands-on Lab: Ensure S3 Buckets Have MFA Delete Enabled to Prevent Deletion Without an MFA Token

# Complete End-to-End Practical Guide

---

# Lab Objective

In this hands-on lab, you will learn how to protect an Amazon S3 bucket from accidental or malicious deletion by enabling **MFA Delete**.

By the end of this lab, you will understand:

* What MFA Delete is
* Why it is important
* How it works internally
* How to enable it using AWS CLI
* How to verify it
* How to test it
* Common errors and troubleshooting
* Production best practices

---

# Important Note Before Starting

## MFA Delete has special restrictions

* ✅ Can **only** be enabled by the **AWS Account Root User**
* ✅ Cannot be enabled from the **AWS Management Console**
* ✅ Must be enabled using the **AWS CLI**
* ✅ Bucket Versioning must already be enabled

If you are logged in with an IAM user, switch to the **Root User** for this lab.

---

# Architecture
<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/4880f2b4-ae7a-4d56-b414-55edfa7a168f" />


---

# Prerequisites

* AWS Account
* Root User Login
* Root User MFA Enabled
* AWS CLI Installed
* AWS CLI Configured
* Region: ap-south-1 (recommended)

---

# Step 1: Verify AWS CLI

```bash
aws --version
```

Expected:

```bash
aws-cli/2.x.x
```

---

# Step 2: Configure AWS CLI

```bash
aws configure
```

Enter:

```text
AWS Access Key ID

AWS Secret Access Key

Region

ap-south-1

Output

json
```

Verify:

```bash
aws sts get-caller-identity
```

Expected:

```json
{
 "Account":"123456789012",
 "Arn":"arn:aws:iam::123456789012:root",
 "UserId":"..."
}
```

The ARN should indicate the **root account**.

---

# Step 3: Create an S3 Bucket

```bash
aws s3 mb s3://mfa-delete-demo-yourname-12345 --region ap-south-1
```

Verify:

```bash
aws s3 ls
```

---

# Step 4: Enable Versioning

```bash
aws s3api put-bucket-versioning \
--bucket mfa-delete-demo-yourname-12345 \
--versioning-configuration Status=Enabled
```

Verify:

```bash
aws s3api get-bucket-versioning \
--bucket mfa-delete-demo-yourname-12345
```

Output:

```json
{
  "Status":"Enabled"
}
```

---

# Why Versioning?

Suppose:

```
resume.pdf
```

gets uploaded.

Someone uploads another version.

Now S3 stores:

```
Version 1

Version 2
```

instead of replacing the file.

MFA Delete protects these versions from permanent deletion.

---

# Step 5: Verify Root MFA Device

Open:

```
AWS Console

↓

Account

↓

Security Credentials

↓

Multi-Factor Authentication
```

Ensure MFA is enabled.

---

# Step 6: Get the MFA Device ARN

Example: 

```text
arn:aws:iam::123456789012:mfa/root-account-mfa-device
```

Keep it ready.  

---

# Step 7: Enable MFA Delete

Current MFA code example:

```
654321
```

Run:

```bash
aws s3api put-bucket-versioning \
--bucket mfa-delete-demo-yourname-12345 \
--versioning-configuration Status=Enabled,MFADelete=Enabled \
--mfa "arn:aws:iam::123456789012:mfa/root-account-mfa-device 654321"
```

Replace:

* ARN with your MFA device ARN
* `654321` with your current MFA code

---

# Step 8: Verify MFA Delete

```bash
aws s3api get-bucket-versioning \
--bucket mfa-delete-demo-yourname-12345
```

Expected:

```json
{
 "Status":"Enabled",
 "MFADelete":"Enabled"
}
```

Congratulations! MFA Delete is enabled.

---

# Step 9: Create a Test File

```bash
echo "Hello AWS" > demo.txt
```

Verify:

```bash
cat demo.txt
```

---

# Step 10: Upload the File

```bash
aws s3 cp demo.txt s3://mfa-delete-demo-yourname-12345/
```

Verify:

```bash
aws s3 ls s3://mfa-delete-demo-yourname-12345
```

---

# Step 11: Find the Version ID

```bash
aws s3api list-object-versions \
--bucket mfa-delete-demo-yourname-12345
```

Output:

```json
{
 "Versions":[
   {
     "Key":"demo.txt",
     "VersionId":"abc123xyz"
   }
 ]
}
```

Copy the VersionId.

---

# Step 12: Try Deleting Without MFA

```bash
aws s3api delete-object \
--bucket mfa-delete-demo-yourname-12345 \
--key demo.txt \
--version-id abc123xyz
```

Expected:

```text
AccessDenied
```
<img width="3342" height="407" alt="image" src="https://github.com/user-attachments/assets/cc549885-3551-44db-81a4-3194ac77f785" />

or

```text
MFA authentication required
```

Deletion is blocked.

---
## Now if you Delete 
### Scenario 1: Delete the latest object from the AWS Console

Suppose your bucket contains:

```test.txt```

In the AWS S3 Console, you select test.txt and click Delete.

Result

### ✅ The delete succeeds.

AWS creates a Delete Marker.

The actual object version is not permanently deleted.

For example:

```bash
Version 1  ---> test.txt
Version 2  ---> Delete Marker
```

<img width="3323" height="1523" alt="image" src="https://github.com/user-attachments/assets/01b4639f-9322-4c0d-91f3-f1d0b0033d1a" />


You can still recover the object by removing the delete marker.

### Scenario 2: Permanently delete a specific version in the AWS Console

If you:

- Open the bucket.
- Enable Show versions.
- Select a specific version.
- Click Delete permanently.
Result

❌ AWS will require MFA.

Without valid MFA, the permanent deletion will not succeed.

This is exactly what you tested with the CLI:

AccessDenied:

Mfa Authentication must be used for this request

<img width="3226" height="651" alt="image" src="https://github.com/user-attachments/assets/96231af1-3687-4eeb-9c26-00425fa580cb" />

### Scenario 3: Suspend Versioning from the AWS Console

If MFA Delete is enabled and you try to suspend versioning:

❌ AWS will not allow the operation without the required MFA authentication.

Summary
- Action	AWS Console Behavior
- Delete an object (latest version)	✅ Creates a Delete Marker
- Restore object	✅ Possible by deleting the Delete Marker
- Permanently delete an object version	❌ Requires MFA
- Disable/Suspend Versioning	❌ Requires MFA
- Permanently delete all versions	❌ Requires MFA
---
# Step 13: Delete With MFA

Generate a fresh MFA code.

Example:

```
345987
```

Run:

```bash
aws s3api delete-object \
--bucket mfa-delete-demo-yourname-12345 \
--key demo.txt \
--version-id abc123xyz \
--mfa "arn:aws:iam::123456789012:mfa/root-account-mfa-device 345987"
```

The delete operation succeeds.

---

# Internal Workflow

```text
Delete Request

       │

       ▼

Amazon S3

       │

Versioning Enabled?

       │

      Yes

       │

MFA Delete Enabled?

       │

      Yes

       │

Validate MFA Token

       │

 ┌─────┴─────┐

 │           │

Valid      Invalid

 │           │

 ▼           ▼

Delete    AccessDenied
```

---

# Real-World Scenario

A ransomware attacker steals an administrator's credentials.

Without MFA Delete:

```
Delete Backup

↓

Success

↓

Backup Lost
```

With MFA Delete:

```
Delete Backup

↓

AWS Requests MFA

↓

Attacker Has No MFA

↓

Delete Fails

↓

Backup Protected
```

---

# Common Errors

## AccessDenied

Cause:

Using an IAM user.

Solution:

Use the **Root User**.

---

## Invalid MFA

Cause:

Expired MFA code.

Solution:

Generate a new code and retry.

---

## Invalid Request

Cause:

Versioning is not enabled.

Solution:

Enable versioning first.

---

## Invalid ARN

Cause:

Wrong MFA device ARN.

Solution:

Copy the ARN exactly from the root account security credentials.

---

## MFA Delete Not Showing

Cause:

The `put-bucket-versioning` command failed or was run by a non-root user.

Solution:

Run it again as the root user with a valid MFA code.

---

# Production Best Practices

* Enable Versioning on critical buckets.
* Enable MFA Delete for highly sensitive buckets.
* Enable Block Public Access.
* Enable SSE-KMS encryption.
* Enable CloudTrail logging.
* Apply least-privilege IAM policies.
* Use S3 Object Lock for immutable backups when compliance requires it.

---

# Limitations

* Only the root account can enable or disable MFA Delete.
* MFA Delete cannot be configured through the AWS Console.
* Automation pipelines cannot easily perform protected delete operations because they cannot provide interactive MFA codes.
* MFA Delete protects version deletion and versioning changes, not every S3 operation.

---

# Cleanup

Delete the remaining object versions using MFA if needed.

Disable MFA Delete (root user only) if you no longer need it.

Delete the bucket:

```bash
aws s3 rb s3://mfa-delete-demo-yourname-12345 --force
```

---

# Interview Questions

## What is MFA Delete?

A feature that requires a valid MFA token before permanently deleting object versions or changing the versioning state of an S3 bucket.

---

## Can an IAM user enable MFA Delete?

No. Only the AWS account root user can enable or disable MFA Delete.

---

## Can MFA Delete be enabled from the AWS Console?

No. It must be configured using the AWS CLI or API.

---

## Is Versioning mandatory?

Yes. MFA Delete only works on version-enabled buckets.

---

## What is a better modern solution for immutable backups?

Use **S3 Object Lock** (Governance or Compliance mode) together with Versioning, SSE-KMS, CloudTrail, and least-privilege IAM policies for stronger protection and regulatory compliance.

---

# Mastery Checklist

* [x] I understand what MFA Delete is.
* [x] I know why Versioning is required.
* [x] I can create an S3 bucket.
* [x] I can enable Versioning.
* [x] I can identify the root MFA device ARN.
* [x] I can enable MFA Delete using the AWS CLI.
* [x] I can verify MFA Delete is enabled.
* [x] I can upload versioned objects.
* [x] I can test deletion with and without MFA.
* [x] I understand when to use MFA Delete versus S3 Object Lock in production.
