
# End-to-End Practical: Ensure Amazon S3 Buckets Are Not Publicly Accessible

# Objective

Secure an Amazon S3 bucket by ensuring that:

* Block Public Access is enabled
* Bucket policies do not allow public access
* ACLs do not grant public permissions
* Only authorized IAM users or roles can access the bucket

This is an AWS Security Hub and CIS AWS Foundations Benchmark recommendation.

---

# Architecture

<img width="1179" height="1334" alt="image" src="https://github.com/user-attachments/assets/0b62d2d1-b608-40f9-a51a-8aeb98e4dc9d" />


---

# Prerequisites

* AWS Account
* IAM user with S3 permissions
* AWS Console access

---

# Step 1: Create an S3 Bucket

Go to:

```
AWS Console
→ S3
→ Create Bucket
```

Bucket Name:

```
secure-data-bucket-demo
```

Region:

```
Asia Pacific (Mumbai)
```

Keep default encryption enabled.

---

# Step 2: Enable Block Public Access

During bucket creation, ensure all four options are checked.

```
☑ Block public ACLs

☑ Ignore public ACLs

☑ Block public bucket policies

☑ Restrict public bucket policies
```

Click **Create Bucket**.

---

# Step 3: Verify Block Public Access

Open the bucket.

Go to

```
Permissions
```

You should see

```
Block Public Access

ON

✓ Block all public access
```

This protects against accidental public exposure.

---

# Step 4: Check Bucket Policy

Go to

```
Permissions

↓

Bucket Policy
```

Initially it should be empty.

```
No bucket policy
```

This is acceptable if access is managed through IAM.

---

# Step 5: Create a Secure Bucket Policy

Example allowing only a specific IAM role:

```json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Sid":"AllowDevOpsRole",
      "Effect":"Allow",
      "Principal":{
        "AWS":"arn:aws:iam::123456789012:role/DevOpsRole"
      },
      "Action":"s3:*",
      "Resource":[
        "arn:aws:s3:::secure-data-bucket-demo",
        "arn:aws:s3:::secure-data-bucket-demo/*"
      ]
    }
  ]
}
```

Click **Save Changes**.

Only this IAM role can access the bucket.

---

# Step 6: Avoid Public Policies

Never use:

```json
{
  "Effect":"Allow",
  "Principal":"*",
  "Action":"s3:GetObject",
  "Resource":"arn:aws:s3:::secure-data-bucket-demo/*"
}
```

Because:

```
Principal = *

means

Everyone on the Internet
```

This makes the bucket public.

---

# Step 7: Check Object ACLs

Open any object.

Go to

```
Permissions
```

Verify:

```
Public Access

Not Public
```

No permissions should be granted to:

```
All Users

Authenticated Users
```

---

# Step 8: Test IAM Access

Assume an IAM user with permission:

```json
{
    "Effect":"Allow",
    "Action":[
        "s3:GetObject",
        "s3:PutObject"
    ],
    "Resource":"arn:aws:s3:::secure-data-bucket-demo/*"
}
```

Upload:

```
report.pdf
```

Download:

```
report.pdf
```

Operations succeed.

---

# Step 9: Test Anonymous Access

Open:

```
https://secure-data-bucket-demo.s3.amazonaws.com/report.pdf
```

Result:

```
Access Denied
```

This confirms the bucket is private.

---

# Step 10: Check Public Access Status

In S3 console:

```
Buckets

↓

secure-data-bucket-demo
```

Status should display:

```
Not Public
```

If it says **Public**, review the bucket policy and Block Public Access settings.

---

# Step 11: Enable Versioning (Recommended)

Go to:

```
Properties

↓

Bucket Versioning

↓

Enable
```

This protects against accidental deletion and overwrites.

---

# Step 12: Enable Default Encryption

Go to:

```
Properties

↓

Default Encryption
```

Choose:

```
SSE-S3

or

SSE-KMS
```

This encrypts objects automatically.

---

# Step 13: Enable Server Access Logging

Go to:

```
Properties

↓

Server Access Logging

↓

Enable
```

Destination bucket:

```
secure-access-logs
```

All access requests will be recorded.

---

# Step 14: Enable Versioning + MFA Delete (Production)

For critical buckets:

```
Versioning

Enabled

+

MFA Delete
```

This provides stronger protection against accidental or malicious deletion.

---

# Step 15: Verify with AWS Trusted Advisor or Security Hub

Open:

```
AWS Security Hub

or

Trusted Advisor
```

You should see:

```
✓ S3 Bucket Not Public

PASS
```

---

# Common Misconfigurations

❌ Principal = "*"

❌ Public Read ACL

❌ Public Write ACL

❌ Block Public Access Disabled

❌ Wildcard permissions for everyone

❌ Allowing anonymous GetObject access

---

# Secure End-to-End Flow

```text
                  Authorized IAM User
                           │
                           ▼
                  IAM Authentication
                           │
                           ▼
                +----------------------+
                | Secure S3 Bucket     |
                +----------------------+
                           │
        +------------------+------------------+
        │                                     │
        ▼                                     ▼
 Block Public Access ON           Private Bucket Policy
        │                                     │
        +------------------+------------------+
                           │
                           ▼
                  Object Stored Securely


        Anonymous User
               │
               ▼

      https://bucket.s3.amazonaws.com/file

               │
               ▼

          ACCESS DENIED
```

---

# Validation Checklist

✅ Bucket created

✅ Block Public Access enabled

✅ Bucket status shows "Not Public"

✅ No public ACLs

✅ No `Principal: "*"` policy

✅ IAM-only access configured

✅ Default encryption enabled

✅ Versioning enabled

✅ Server Access Logging enabled

✅ Anonymous access returns "Access Denied"

---

# Interview Question

**Q: How do you ensure an S3 bucket is not publicly accessible?**

**Answer:**

* Enable all four **Block Public Access** settings.
* Avoid bucket policies with `Principal: "*"`.
* Remove public ACLs.
* Grant access only through IAM users or IAM roles.
* Enable encryption, versioning, and server access logging.
* Regularly verify the bucket status using AWS Security Hub or Trusted Advisor.
