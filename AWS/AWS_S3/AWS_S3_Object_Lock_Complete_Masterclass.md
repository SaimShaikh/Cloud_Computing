# AWS S3 Object Lock -- Complete Masterclass

------------------------------------------------------------------------

# 1. Introduction

Amazon S3 Object Lock is an S3 feature that protects objects from being
deleted or overwritten for a specified period of time or indefinitely
using Legal Hold.

It enables **WORM (Write Once Read Many)** storage and is commonly used
for compliance, ransomware protection, backups, financial records,
healthcare data, and legal archives.

------------------------------------------------------------------------

# 2. Why Object Lock Exists

Problems solved:

-   Accidental deletion
-   Malicious administrators
-   Ransomware attacks
-   Regulatory compliance
-   Immutable backups

------------------------------------------------------------------------

# 3. WORM

    Upload Object
          │
          ▼
    Stored
          │
          ▼
    Cannot Modify
    Cannot Delete
    Until Retention Expires

------------------------------------------------------------------------

# 4. Requirements

-   Bucket must be created **with Object Lock enabled**
-   Bucket Versioning is required
-   Existing buckets cannot have Object Lock enabled later
-   IAM permissions required

------------------------------------------------------------------------

# 5. Modes

## Governance Mode

-   Users with special IAM permission can bypass.
-   Suitable for internal governance.
-   Testing retention periods, preventing accidental changes while keeping an "escape hatch".
-   users with s3:BypassGovernanceRetention can delete or alter the lock.

Permission:

    s3:BypassGovernanceRetention

------------------------------------------------------------------------

## Compliance Mode

-   Nobody can bypass.
-   Root user cannot delete objects.
-   AWS Support cannot remove protection.
-   Retention days can only be extended; they can never be reduced or removed.

Recommended for regulated industries.

------------------------------------------------------------------------

# 6. Governance vs Compliance

  Feature            Governance              Compliance
  ------------------ ----------------------- ------------
  Admin can bypass   Yes                     No
  Root can bypass    Yes (with permission)   No
  AWS Support        No                      No
  Compliance Ready   Limited                 Yes

------------------------------------------------------------------------

# 7. Retention

Retention specifies how long an object is protected.

Example:

    Upload
      │
    365 Days
      │
    Delete?
      │
    Denied

------------------------------------------------------------------------

# 8. Legal Hold

Legal Hold has no expiration.

    Enable Legal Hold
          │
    Object Protected
          │
    Remove Legal Hold
          │
    Deletion Possible

------------------------------------------------------------------------

# 9. Hands-on (AWS Console)

## Lab 1

Create Bucket

-   Open S3
-   Create Bucket
-   Check **Enable Object Lock**
-   Create bucket

Versioning is enabled automatically.

------------------------------------------------------------------------

## Lab 2

Upload Object

Upload:

    sample.txt

------------------------------------------------------------------------

## Lab 3

Apply Governance Mode

Object

→ Properties

→ Object Lock

→ Governance

Retention

7 Days

Save

Try deleting before 7 days.

Expected:

    Access Denied

------------------------------------------------------------------------

## Lab 4

Compliance Mode

Repeat upload.

Apply

Compliance Mode

Retention

7 Days

Try deleting.

Result:

    Delete Denied

------------------------------------------------------------------------

## Lab 5

Legal Hold

Object

Properties

Object Lock

Legal Hold

Enable

Deletion will fail until disabled.

------------------------------------------------------------------------

# 10. AWS CLI

Create bucket:

``` bash
aws s3api create-bucket \
--bucket my-object-lock-demo \
--region ap-south-1 \
--object-lock-enabled-for-bucket
```

Upload:

``` bash
aws s3 cp test.txt s3://my-object-lock-demo/
```

Enable retention:

``` bash
aws s3api put-object-retention \
--bucket my-object-lock-demo \
--key test.txt \
--retention Mode=GOVERNANCE,RetainUntilDate=2026-12-31T00:00:00Z
```

Enable Legal Hold:

``` bash
aws s3api put-object-legal-hold \
--bucket my-object-lock-demo \
--key test.txt \
--legal-hold Status=ON
```

------------------------------------------------------------------------

# 11. Terraform

``` hcl
resource "aws_s3_bucket" "demo" {
  bucket = "my-object-lock-demo"

  object_lock_enabled = true
}

resource "aws_s3_bucket_versioning" "versioning" {
  bucket = aws_s3_bucket.demo.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

------------------------------------------------------------------------

# 12. IAM Permission

To bypass Governance Mode:

``` json
{
  "Effect":"Allow",
  "Action":"s3:BypassGovernanceRetention",
  "Resource":"*"
}
```

------------------------------------------------------------------------

# 13. Lifecycle

Lifecycle expiration does not delete an object protected by Object Lock
before its retention period ends.

------------------------------------------------------------------------

# 14. Replication

Supports Cross-Region Replication when destination bucket is also
configured for Object Lock.

------------------------------------------------------------------------

# 15. CloudTrail

Monitor:

-   PutObjectRetention
-   PutObjectLegalHold
-   DeleteObject
-   PutObjectLockConfiguration

------------------------------------------------------------------------

# 16. Cost

Object Lock has **no additional feature charge**.

You pay only for:

-   S3 Storage
-   Requests
-   Versioned objects
-   Replication (if enabled)

------------------------------------------------------------------------

# 17. Best Practices

-   Use Compliance Mode for regulated data.
-   Enable bucket versioning.
-   Test Governance Mode before production.
-   Enable CloudTrail.
-   Restrict bypass permission.

------------------------------------------------------------------------

# 18. Edge Cases

-   Cannot enable Object Lock on an existing bucket.
-   Cannot disable Object Lock after bucket creation.
-   Multiple object versions each have independent retention.
-   Lifecycle cannot override retention.

------------------------------------------------------------------------

# 19. Troubleshooting

**Delete fails**

Cause:

-   Retention active
-   Legal Hold enabled
-   Missing bypass permission

------------------------------------------------------------------------

# 20. Comparison

  Feature           Object Lock   Glacier Vault Lock
  ----------------- ------------- ------------------------
  Service           Amazon S3     Legacy Glacier Vault
  Bucket Support    Yes           No
  WORM              Yes           Yes
  Legal Hold        Yes           No
  Governance Mode   Yes           No
  Compliance Mode   Yes           Permanent Vault Policy

------------------------------------------------------------------------

# 21. Interview Questions

1.  What is Object Lock?
2.  Why is versioning mandatory?
3.  Difference between Governance and Compliance?
4.  Can the root user delete Compliance Mode objects?
5.  What is Legal Hold?
6.  Can Lifecycle delete locked objects?
7.  Difference between Object Lock and MFA Delete?
8.  Why is Object Lock used against ransomware?
9.  Can Object Lock be enabled on an existing bucket?
10. Difference between Glacier Vault Lock and Object Lock?

------------------------------------------------------------------------

# 22. Cleanup

-   Remove Legal Hold
-   Wait for retention expiry (or bypass Governance if authorized)
-   Delete object versions
-   Delete bucket

------------------------------------------------------------------------

# Summary

Object Lock provides immutable object storage for Amazon S3 using WORM
principles. It protects against accidental deletion, ransomware, and
compliance violations using Governance Mode, Compliance Mode, retention
periods, and Legal Hold. It is the recommended modern replacement for
legacy Glacier Vault Lock use cases.
