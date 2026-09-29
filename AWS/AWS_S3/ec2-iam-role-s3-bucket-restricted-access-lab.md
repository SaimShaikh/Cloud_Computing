
# EC2 + IAM Role — S3 Bucket-Restricted Access Lab (AWS Console Only)

## 1. Objective

Create an EC2 instance and attach an IAM role that allows the instance to:

- List **only one specific S3 bucket**
- Download objects **only from that bucket**
- Access another S3 bucket: **Denied**

This lab is completely **AWS Management Console based**. No AWS CLI commands are required.

---

# 2. Architecture

```text
                         AWS Account
                              |
                    +---------+---------+
                    |                   |
              S3 Bucket A          S3 Bucket B
               ALLOWED              DENIED
                    |                   |
                    +---------+---------+
                              |
                       IAM Role
                EC2-S3-Restricted-Role
                              |
                       Instance Profile
                              |
                            EC2
```

Expected result:

```text
EC2
 |
 +----> Allowed Bucket
 |       List   = ALLOWED
 |       Get    = ALLOWED
 |
 +----> Denied Bucket
         List   = DENIED
         Get    = DENIED
```

---

# 3. Prerequisites

You need:

- AWS account
- Permission to create EC2, IAM, and S3 resources
- AWS Management Console access
- Region: **Asia Pacific (Mumbai) — ap-south-1**

---

# 4. Lab Resources

Create these resources:

| Resource | Name |
|---|---|
| Allowed S3 bucket | `ec2-iam-allowed-bucket-<unique>` |
| Denied S3 bucket | `ec2-iam-denied-bucket-<unique>` |
| IAM policy | `EC2-S3-Restricted-Policy` |
| IAM role | `EC2-S3-Restricted-Role` |
| EC2 instance | `EC2-S3-Test` |

S3 bucket names must be globally unique.

Example:

```text
ec2-iam-allowed-bucket-saime-2026
ec2-iam-denied-bucket-saime-2026
```

---

# 5. Step 1 — Create the Allowed S3 Bucket

Go to:

```text
AWS Console
→ S3
→ General purpose buckets
→ Create bucket
```

Enter:

```text
Bucket name:
ec2-iam-allowed-bucket-<unique>
```

Region:

```text
Asia Pacific (Mumbai) ap-south-1
```

For this lab, keep the default settings unless your account requires otherwise.

Click:

```text
Create bucket
```

---

# 6. Step 2 — Create the Denied S3 Bucket

Again:

```text
S3
→ General purpose buckets
→ Create bucket
```

Bucket name:

```text
ec2-iam-denied-bucket-<unique>
```

Region:

```text
Asia Pacific (Mumbai) ap-south-1
```

Click:

```text
Create bucket
```

You should now have:

```text
ec2-iam-allowed-bucket-<unique>
ec2-iam-denied-bucket-<unique>
```

---

# 7. Step 3 — Upload Test Objects

## Upload object to Allowed Bucket

Open:

```text
S3
→ ec2-iam-allowed-bucket-<unique>
→ Objects
→ Upload
```

Create a local text file:

```text
allowed.txt
```

Content:

```text
This object belongs to the allowed bucket.
```

Upload it.

---

## Upload object to Denied Bucket

Open:

```text
S3
→ ec2-iam-denied-bucket-<unique>
→ Objects
→ Upload
```

Upload:

```text
denied.txt
```

Content:

```text
This object belongs to the denied bucket.
```

Now both buckets contain a test object.

---

# 8. Step 4 — Create the IAM Policy

Go to:

```text
IAM
→ Policies
→ Create policy
```

Select:

```text
JSON
```

Enter:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListOnlyAllowedBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::ec2-iam-allowed-bucket-<unique>"
    },
    {
      "Sid": "DownloadOnlyFromAllowedBucket",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::ec2-iam-allowed-bucket-<unique>/*"
    }
  ]
}
```

Replace:

```text
ec2-iam-allowed-bucket-<unique>
```

with your actual allowed bucket name.

Example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListOnlyAllowedBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::ec2-iam-allowed-bucket-saime-2026"
    },
    {
      "Sid": "DownloadOnlyFromAllowedBucket",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::ec2-iam-allowed-bucket-saime-2026/*"
    }
  ]
}
```

Click:

```text
Next
```

Review the policy.

Policy name:

```text
EC2-S3-Restricted-Policy
```

Description:

```text
Allows EC2 to list and download objects only from the allowed S3 bucket.
```

Click:

```text
Create policy
```

---

# 9. Understand the Policy Before Continuing

There are two permissions.

## Permission 1 — List the bucket

```json
"Action": "s3:ListBucket"
```

Resource:

```text
arn:aws:s3:::ALLOWED-BUCKET
```

This allows the EC2 instance to see/list objects in that bucket.

---

## Permission 2 — Download objects

```json
"Action": "s3:GetObject"
```

Resource:

```text
arn:aws:s3:::ALLOWED-BUCKET/*
```

The `/*` means objects inside the bucket.

---

# 10. Step 5 — Create the IAM Role

Go to:

```text
IAM
→ Roles
→ Create role
```

Under:

```text
Trusted entity type
```

Select:

```text
AWS service
```

Under:

```text
Service or use case
```

Select:

```text
EC2
```

This creates the trust relationship:

```text
EC2
 |
 | AssumeRole
 v
IAM Role
```

Click:

```text
Next
```

---

# 11. Step 6 — Attach the S3 Policy to the Role

In:

```text
Add permissions
```

Search:

```text
EC2-S3-Restricted-Policy
```

Select the policy.

Click:

```text
Next
```

---

# 12. Step 7 — Name the Role

Role name:

```text
EC2-S3-Restricted-Role
```

Description:

```text
Allows EC2 to list and download objects only from one S3 bucket.
```

Click:

```text
Create role
```

The role now contains:

```text
Trust policy:
    EC2 service

Permissions policy:
    EC2-S3-Restricted-Policy
```

---

# 13. Step 8 — Verify the IAM Role

Open:

```text
IAM
→ Roles
→ EC2-S3-Restricted-Role
```

Check:

```text
Permissions
```

You should see:

```text
EC2-S3-Restricted-Policy
```

Then open:

```text
Trust relationships
```

You should see EC2 as the trusted service.

Conceptually:

```text
EC2
 |
 | sts:AssumeRole
 v
EC2-S3-Restricted-Role
 |
 +---- s3:ListBucket
 |
 +---- s3:GetObject
```

---

# 14. Step 9 — Launch the EC2 Instance

Go to:

```text
EC2
→ Instances
→ Launch instances
```

Name:

```text
EC2-S3-Test
```

AMI:

```text
Ubuntu Server 24.04 LTS
```

Instance type:

```text
t3.micro
```

Select/create your key pair as required.

Configure networking according to your environment.

For the IAM role:

Go to:

```text
Advanced details
→ IAM instance profile
```

Select:

```text
EC2-S3-Restricted-Role
```

Depending on the console version, the selector may display the associated instance profile rather than the role name.

Launch the instance.

---

# 15. Step 10 — Verify the IAM Role Is Attached

After the instance is running:

```text
EC2
→ Instances
→ EC2-S3-Test
```

Open:

```text
Security
```

Look for:

```text
IAM Role
```

You should see:

```text
EC2-S3-Restricted-Role
```

This confirms that the EC2 instance has the role.

---

# 16. Step 11 — Connect to EC2

From:

```text
EC2
→ Instances
→ EC2-S3-Test
```

Click:

```text
Connect
```

You can use:

```text
EC2 Instance Connect
```

if supported by the instance and configuration.

Open the terminal.

---

# 17. Step 12 — Install AWS CLI Only If Needed

For this lab, you do **not** need to configure an AWS access key.

The purpose is to test the IAM role attached to EC2.

If you use a terminal-based test, make sure the AWS CLI is using the instance role rather than a manually configured IAM user's access keys.

The important identity to verify is:

```text
EC2-S3-Restricted-Role
```

---

# 18. Step 13 — Test Access to Allowed Bucket

From the EC2 terminal, test listing the allowed bucket.

Expected:

```text
allowed.txt
```

Then test downloading:

```text
allowed.txt
```

Expected:

```text
This object belongs to the allowed bucket.
```

Result:

```text
LIST  → ALLOWED
GET   → ALLOWED
```

---

# 19. Step 14 — Test Access to Denied Bucket

Now try to list the denied bucket.

Expected:

```text
AccessDenied
```

Then try to download:

```text
denied.txt
```

Expected:

```text
AccessDenied
```

Result:

```text
LIST  → DENIED
GET   → DENIED
```

---

# 20. Important Testing Note

The AWS Console itself is logged in as **your IAM user/role**, not as the EC2 instance role.

Therefore:

```text
S3 Console
    |
    +---- uses your console identity

EC2
    |
    +---- uses EC2-S3-Restricted-Role
```

To prove the EC2 role restriction, perform the S3 access test **from inside the EC2 instance**.

Do not simply open S3 from the AWS Console and assume that proves the EC2 role works.

---

# 21. Expected Result

| Test | Result |
|---|---|
| EC2 → List Allowed Bucket | ALLOWED |
| EC2 → Download Allowed Object | ALLOWED |
| EC2 → List Denied Bucket | DENIED |
| EC2 → Download Denied Object | DENIED |

## With Allow 
<img width="1680" height="1050" alt="Screenshot 2026-09-28 at 11 50 50 AM" src="https://github.com/user-attachments/assets/6086d3cf-fd26-41ea-862a-cb696022a088" />

## With Deny 
<img width="1144" height="199" alt="image" src="https://github.com/user-attachments/assets/3a6e62a3-79dc-4e2e-81e9-ef22a22a9dd7" />

Architecture:

```text
                    EC2
                     |
             IAM Instance Profile
                     |
          EC2-S3-Restricted-Role
                     |
             IAM Permission Policy
                     |
          +----------+----------+
          |                     |
    Allowed Bucket        Denied Bucket
          |                     |
       LIST ✓                  LIST ✗
       GET  ✓                  GET  ✗
```

---

# 22. Why Does the Denied Bucket Fail?

There is no permission in the IAM role for:

```text
ec2-iam-denied-bucket-<unique>
```

The role only allows:

```text
s3:ListBucket
```

on:

```text
Allowed Bucket
```

and:

```text
s3:GetObject
```

on:

```text
Allowed Bucket/*
```

Therefore, the denied bucket has no applicable Allow.

---

# 23. Important IAM Concept — Explicit Deny

This lab does not require an explicit `Deny`.

AWS authorization generally works like:

```text
Request
   |
   v
Is there an applicable Allow?
   |
   +---- YES ---> Continue
   |
   +---- NO ----> Access Denied
```

If an applicable explicit `Deny` exists:

```text
Explicit Deny
      |
      v
Overrides Allow
```

---

# 24. Why `ListBucket` and `GetObject` Are Different

This is an important interview concept.

### List

```text
s3:ListBucket
```

Resource:

```text
arn:aws:s3:::bucket-name
```

### Download

```text
s3:GetObject
```

Resource:

```text
arn:aws:s3:::bucket-name/*
```

So this is wrong for `GetObject`:

```text
arn:aws:s3:::bucket-name
```

Correct:

```text
arn:aws:s3:::bucket-name/*
```

---

# 25. Why Use IAM Role Instead of Access Keys?

Do not store long-term AWS access keys inside:

```text
EC2
Application code
.env files
Git repositories
Docker images
```

Instead:

```text
EC2
 |
 | Instance Profile
 v
IAM Role
 |
 | Temporary Credentials
 v
AWS Services
```

The EC2 role provides temporary credentials to applications running on the instance.

---

# 26. Common Mistakes

## Mistake 1 — Giving `s3:*`

Avoid:

```text
s3:*
```

when only list/download are required.

---

## Mistake 2 — Using `Resource: "*"`

Avoid:

```text
"Resource": "*"
```

when only one bucket is required.

---

## Mistake 3 — Using bucket ARN for `GetObject`

Wrong:

```text
arn:aws:s3:::bucket-name
```

Correct:

```text
arn:aws:s3:::bucket-name/*
```

---

## Mistake 4 — Testing from the S3 Console

The S3 Console uses your console identity.

It does not automatically use the EC2 instance role.

Test the role from the EC2 instance.

---

## Mistake 5 — EC2 Has Another IAM Role

Check:

```text
EC2
→ Instance
→ Security
→ IAM Role
```

Make sure it is:

```text
EC2-S3-Restricted-Role
```

---

