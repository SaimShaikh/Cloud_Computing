# EC2 + IAM Role — S3 Bucket-Restricted Access Lab

## 1. Objective

Create an EC2 instance and attach an IAM role that allows the instance to:

- List **only one specific S3 bucket**
- Download objects **only from that bucket**
- Access another S3 bucket: **Denied**

This lab demonstrates **EC2 Instance Profile + IAM Role + least-privilege S3 permissions**.

---

## 2. Architecture

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
                              |
                         AWS CLI
```

Expected result:

```text
EC2
 |
 +----> Bucket A
 |       List   = ALLOWED
 |       Get    = ALLOWED
 |
 +----> Bucket B
         List   = DENIED
         Get    = DENIED
```

---

# 3. Prerequisites

- AWS account
- Permission to create EC2, IAM roles/policies, and S3 resources
- AWS CLI
- SSH access to the EC2 instance
- Region: `ap-south-1` (Mumbai)

---

# 4. Lab Variables

Use unique S3 bucket names.

```bash
REGION="ap-south-1"

ALLOWED_BUCKET="ec2-iam-allowed-bucket-<unique>"
DENIED_BUCKET="ec2-iam-denied-bucket-<unique>"

ROLE_NAME="EC2-S3-Restricted-Role"
INSTANCE_PROFILE="EC2-S3-Restricted-InstanceProfile"
POLICY_NAME="EC2-S3-Restricted-Policy"
```

Example:

```text
ec2-iam-allowed-bucket-saime-2026
ec2-iam-denied-bucket-saime-2026
```

S3 bucket names must be globally unique.

---

# 5. Create Two S3 Buckets

Create the allowed bucket:

```bash
aws s3api create-bucket \
  --bucket "$ALLOWED_BUCKET" \
  --region "$REGION" \
  --create-bucket-configuration LocationConstraint="$REGION"
```

Create the denied bucket:

```bash
aws s3api create-bucket \
  --bucket "$DENIED_BUCKET" \
  --region "$REGION" \
  --create-bucket-configuration LocationConstraint="$REGION"
```

Verify:

```bash
aws s3 ls
```

---

# 6. Upload Test Objects

Create test files:

```bash
echo "This object belongs to the allowed bucket." > allowed.txt
echo "This object belongs to the denied bucket." > denied.txt
```

Upload:

```bash
aws s3 cp allowed.txt s3://$ALLOWED_BUCKET/
aws s3 cp denied.txt s3://$DENIED_BUCKET/
```

Verify:

```bash
aws s3 ls s3://$ALLOWED_BUCKET/
aws s3 ls s3://$DENIED_BUCKET/
```

---

# 7. Create the IAM Permission Policy

Create:

```bash
nano ec2-s3-restricted-policy.json
```

Use:

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

Replace `<unique>` with your actual bucket name.

### Why two statements?

`ListBucket` applies to the bucket:

```text
arn:aws:s3:::bucket-name
```

`GetObject` applies to objects:

```text
arn:aws:s3:::bucket-name/*
```

Do **not** use:

```json
"Action": "s3:*",
"Resource": "*"
```

for this lab.

---

# 8. Create the IAM Policy

```bash
aws iam create-policy \
  --policy-name "$POLICY_NAME" \
  --policy-document file://ec2-s3-restricted-policy.json
```

Get the policy ARN:

```bash
aws iam list-policies \
  --scope Local \
  --query 'Policies[?PolicyName==`EC2-S3-Restricted-Policy`].Arn' \
  --output text
```

Example:

```text
arn:aws:iam::123456789012:policy/EC2-S3-Restricted-Policy
```

---

# 9. Create the IAM Role

Create the trust policy:

```bash
nano ec2-trust-policy.json
```

Add:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Create the role:

```bash
aws iam create-role \
  --role-name "$ROLE_NAME" \
  --assume-role-policy-document file://ec2-trust-policy.json
```

The trust policy means:

```text
EC2 service
    |
    | sts:AssumeRole
    v
EC2-S3-Restricted-Role
```

---

# 10. Attach the S3 Policy to the Role

Replace `<ACCOUNT-ID>`:

```bash
aws iam attach-role-policy \
  --role-name "$ROLE_NAME" \
  --policy-arn arn:aws:iam::<ACCOUNT-ID>:policy/$POLICY_NAME
```

Verify:

```bash
aws iam list-attached-role-policies \
  --role-name "$ROLE_NAME"
```

---

# 11. Create the EC2 Instance Profile

EC2 receives the IAM role through an **instance profile**.

Create it:

```bash
aws iam create-instance-profile \
  --instance-profile-name "$INSTANCE_PROFILE"
```

Add the role:

```bash
aws iam add-role-to-instance-profile \
  --instance-profile-name "$INSTANCE_PROFILE" \
  --role-name "$ROLE_NAME"
```

Verify:

```bash
aws iam get-instance-profile \
  --instance-profile-name "$INSTANCE_PROFILE"
```

---

# 12. Launch the EC2 Instance

Example:

```text
AMI: Ubuntu Server 24.04 LTS
Instance type: t3.micro
Region: ap-south-1
```

Attach:

```text
EC2-S3-Restricted-InstanceProfile
```

from:

```text
EC2 Console
→ Launch Instance
→ Advanced Details
→ IAM Instance Profile
```

You can also attach the role to an existing instance.

---

# 13. Verify the Role from EC2

SSH into the instance.

For IMDSv2:

```bash
TOKEN=$(curl -X PUT \
  "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
```

Get the attached role:

```bash
curl \
  -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

Expected:

```text
EC2-S3-Restricted-Role
```

---

# 14. Verify AWS CLI Uses the EC2 Role

Run:

```bash
aws sts get-caller-identity
```

Expected ARN:

```text
arn:aws:sts::<ACCOUNT-ID>:assumed-role/EC2-S3-Restricted-Role/...
```

Also check:

```bash
aws configure list
```

The CLI should use the EC2 role's temporary credentials, not manually configured long-term access keys.

---

# 15. Test 1 — List Allowed Bucket

```bash
aws s3 ls s3://$ALLOWED_BUCKET/
```

Expected:

```text
allowed.txt
```

Result:

```text
SUCCESS
```

Reason:

```text
s3:ListBucket
```

is allowed on the allowed bucket.

---

# 16. Test 2 — Download from Allowed Bucket

```bash
aws s3 cp \
  s3://$ALLOWED_BUCKET/allowed.txt \
  ./allowed-downloaded.txt
```

Verify:

```bash
cat allowed-downloaded.txt
```

Expected:

```text
This object belongs to the allowed bucket.
```

Result:

```text
SUCCESS
```

Reason:

```text
s3:GetObject
```

is allowed for objects inside the allowed bucket.

---

# 17. Test 3 — List Denied Bucket

```bash
aws s3 ls s3://$DENIED_BUCKET/
```

Expected:

```text
An error occurred (AccessDenied) when calling the ListObjectsV2 operation:
Access Denied
```

Result:

```text
DENIED
```

There is no applicable `s3:ListBucket` Allow for this bucket.

---

# 18. Test 4 — Download from Denied Bucket

```bash
aws s3 cp \
  s3://$DENIED_BUCKET/denied.txt \
  ./denied-downloaded.txt
```

Expected:

```text
AccessDenied
```

Result:

```text
DENIED
```

There is no applicable `s3:GetObject` Allow for this bucket.

---

# 19. Final Test Matrix

| Operation | Allowed Bucket | Denied Bucket |
|---|---|---|
| List bucket | ALLOWED | DENIED |
| Download object | ALLOWED | DENIED |
| `s3:ListBucket` | Yes | No |
| `s3:GetObject` | Yes | No |

---

# 20. Important IAM Concept

The policy is restricted by both **Action** and **Resource**.

```text
s3:ListBucket
        |
        +----> arn:aws:s3:::ALLOWED-BUCKET

s3:GetObject
        |
        +----> arn:aws:s3:::ALLOWED-BUCKET/*
```

This is the principle of **least privilege**.

---

# 21. Why `ListBucket` and `GetObject` Are Separate

### `s3:ListBucket`

Controls listing objects:

```text
arn:aws:s3:::bucket-name
```

### `s3:GetObject`

Controls reading/downloading objects:

```text
arn:aws:s3:::bucket-name/*
```

Therefore this is wrong:

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::bucket-name"
}
```

Correct:

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::bucket-name/*"
}
```

---

# 22. Why Use an IAM Role Instead of Access Keys?

Avoid storing long-term AWS access keys in:

```text
EC2 source code
.env files
shell scripts
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
 | Temporary credentials
 v
S3
```

The AWS CLI and applications on EC2 can obtain temporary credentials automatically.

---

# 23. Important: Role vs Instance Profile

They are related but not the same thing.

```text
IAM Role
    |
    | permissions + trust policy
    v
Instance Profile
    |
    | attached to EC2
    v
EC2
```

The instance profile is the EC2 mechanism used to associate the role with the instance.

---

# 24. Common Mistakes

## Mistake 1 — Wrong `GetObject` Resource

Wrong:

```text
arn:aws:s3:::bucket-name
```

Correct:

```text
arn:aws:s3:::bucket-name/*
```

## Mistake 2 — Giving `s3:*`

Avoid:

```text
s3:*
```

when only listing and downloading are required.

## Mistake 3 — Using `Resource: "*"`

Avoid broad resource access when the requirement is one bucket.

## Mistake 4 — Forgetting `ListBucket`

If you want:

```bash
aws s3 ls s3://bucket-name/
```

you need:

```text
s3:ListBucket
```

## Mistake 5 — Testing with a different credential

If `aws configure` has an access key configured on the EC2 instance, you might accidentally test the IAM user's permissions instead of the EC2 role.

Verify:

```bash
aws sts get-caller-identity
```

---

# 25. Advanced Verification — IAM Policy Simulation

You can test the role without actually accessing S3.

Allowed bucket:

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::<ACCOUNT-ID>:role/$ROLE_NAME \
  --action-names s3:ListBucket \
  --resource-arns arn:aws:s3:::$ALLOWED_BUCKET
```

Expected decision:

```text
allowed
```

Denied bucket:

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::<ACCOUNT-ID>:role/$ROLE_NAME \
  --action-names s3:ListBucket \
  --resource-arns arn:aws:s3:::$DENIED_BUCKET
```

Expected decision:

```text
implicitDeny
```

---

# 26. Why Is the Denied Bucket Denied?

Our policy does not need an explicit `Deny`.

There is simply no applicable `Allow` for Bucket B.

Conceptually:

```text
Request
   |
   v
Applicable Allow?
   |
   +---- YES ---> Access can proceed
   |
   +---- NO ----> Access Denied
```

An explicit `Deny` from another applicable policy layer would override an `Allow`.

---

# 27. Other Policy Layers to Know

In a real AWS environment, access can be affected by multiple controls:

```text
IAM identity policy
        +
S3 bucket policy
        +
SCP
        +
VPC endpoint policy
        +
KMS key policy
        +
Other applicable controls
```

For this lab, the main restriction comes from the EC2 role's identity policy.

If the object uses SSE-KMS, additional KMS permissions may be required.

---

# 28. Optional Extension — Add Upload Permission

If you later want the EC2 instance to upload objects too, add:

```json
{
  "Sid": "UploadOnlyToAllowedBucket",
  "Effect": "Allow",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::ec2-iam-allowed-bucket-<unique>/*"
}
```

Then test:

```bash
echo "Uploaded from EC2" > upload-test.txt

aws s3 cp   upload-test.txt   s3://$ALLOWED_BUCKET/
```

Do not add this permission if the application only needs read access.

---

# 29. Cleanup

Delete objects:

```bash
aws s3 rm s3://$ALLOWED_BUCKET/allowed.txt
aws s3 rm s3://$DENIED_BUCKET/denied.txt
```

Delete buckets:

```bash
aws s3 rb s3://$ALLOWED_BUCKET
aws s3 rb s3://$DENIED_BUCKET
```

Detach the policy:

```bash
aws iam detach-role-policy   --role-name "$ROLE_NAME"   --policy-arn arn:aws:iam::<ACCOUNT-ID>:policy/$POLICY_NAME
```

Remove the role from the instance profile:

```bash
aws iam remove-role-from-instance-profile   --instance-profile-name "$INSTANCE_PROFILE"   --role-name "$ROLE_NAME"
```

Delete instance profile:

```bash
aws iam delete-instance-profile   --instance-profile-name "$INSTANCE_PROFILE"
```

Delete role:

```bash
aws iam delete-role   --role-name "$ROLE_NAME"
```

Delete policy:

```bash
aws iam delete-policy   --policy-arn arn:aws:iam::<ACCOUNT-ID>:policy/$POLICY_NAME
```

Terminate the EC2 instance from:

```text
EC2 Console
→ Instances
→ Select instance
→ Terminate instance
```

---

# 30. Interview Questions

### Q1. Why does `s3:ListBucket` use the bucket ARN?

Because listing is an operation on the bucket:

```text
arn:aws:s3:::bucket-name
```

### Q2. Why does `s3:GetObject` use `bucket/*`?

Because `GetObject` operates on individual objects:

```text
arn:aws:s3:::bucket-name/*
```

### Q3. Why use an IAM role on EC2?

It provides temporary credentials without storing long-term access keys on the server.

### Q4. Can this EC2 instance access another S3 bucket?

Not through this role, because the role has no applicable Allow for that bucket.

### Q5. Does the role need `s3:PutObject`?

No. The requirement is only list and download.

### Q6. Does `s3:ListBucket` allow downloading objects?

No. `s3:ListBucket` and `s3:GetObject` are separate permissions.

### Q7. What happens if an explicit Deny exists?

An explicit Deny overrides an Allow.

### Q8. Can multiple EC2 instances use the same IAM role?

Yes. Every instance using the role receives the same role permissions.

### Q9. What is least privilege here?

```text
Actions:
    s3:ListBucket
    s3:GetObject

Resources:
    One specific bucket
    Objects inside that bucket
```

---

# 31. Final Expected Result

```text
                         EC2
                          |
                 Instance Profile
                          |
              EC2-S3-Restricted-Role
                          |
               +----------+----------+
               |                     |
          Allowed Bucket        Denied Bucket
               |                     |
          List = ALLOW          List = DENY
          Get  = ALLOW          Get  = DENY
```

## Key AWS Lesson

> Grant the smallest required actions on the smallest required resources.

For this lab:

```text
s3:ListBucket
    -> one specific bucket

s3:GetObject
    -> objects inside that same bucket
```
