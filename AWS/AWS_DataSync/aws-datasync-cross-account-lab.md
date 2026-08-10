# AWS DataSync — Cross-Account S3-to-S3 Migration Lab

*Verified against AWS's official cross-account S3 tutorial (docs.aws.amazon.com/datasync).*

## Scenario

You have two AWS accounts:

- **Account A (Source)** — owns the data, e.g. `source-bucket-a`
- **Account B (Destination)** — receives the data, e.g. `dest-bucket-b`

You need to move (and optionally keep syncing) objects from Account A's S3 bucket into Account B's S3 bucket, without making either bucket public, and with the transfer running as a managed, resumable, verifiable AWS DataSync task.

> **Confirmed directly from AWS docs:** cross-account S3 *locations* cannot be created from the DataSync console — only the destination location step requires CloudShell/CLI. Everything else (role, bucket policy, task) is console.

---

## Architecture

```
┌─────────────────────────────┐        ┌─────────────────────────────┐
│        ACCOUNT A (Source)    │        │      ACCOUNT B (Destination) │
│                              │        │                              │
│  ┌────────────────────┐      │        │      ┌────────────────────┐  │
│  │  source-bucket-a    │      │        │      │   dest-bucket-b     │  │
│  │  (owns the data)    │      │        │      │   (receives data)   │  │
│  └─────────┬──────────┘      │        │      └─────────▲──────────┘  │
│            │                  │        │                │             │
│  ┌─────────▼──────────┐      │        │      ┌──────────┴─────────┐  │
│  │  DataSync Task      │──────┼────────┼─────▶│  Bucket Policy      │  │
│  │  (runs HERE, in     │      │  writes │      │  trusts source-     │  │
│  │  same Region as     │      │  via    │      │  datasync-role       │  │
│  │  destination         │      │  role   │      │  by exact ARN        │  │
│  │  location — see      │      │  assume │      └─────────────────────┘  │
│  │  Step 5 warning)     │      │         │                              │
│  │                     │      │         │      Object Ownership:        │
│  │  IAM Role:          │      │         │      ACLs disabled            │
│  │  source-datasync-role│      │         │      (bucket owner enforced) │
│  └────────────────────┘      │        │                              │
└─────────────────────────────┘        └─────────────────────────────┘
```

**Key mental model:** the DataSync task and *both* locations are created in the **source account**. The destination account creates zero DataSync resources — it only grants trust via bucket policy (and optionally KMS key policy).

---

## Prerequisites

### Source account (Account A) — user permissions

The IAM identity you're using in the console (you, not the DataSync role) needs permission to create DataSync locations/tasks **and** to create the IAM role/policy manually, since DataSync's "auto-generate role" option only works for same-account transfers. At minimum:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SourceUserRolePermissions",
      "Effect": "Allow",
      "Action": [
        "datasync:CreateLocationS3",
        "datasync:CreateTask",
        "datasync:DescribeLocation*",
        "datasync:DescribeTaskExecution",
        "datasync:ListLocations",
        "datasync:ListTaskExecutions",
        "datasync:DescribeTask",
        "datasync:CancelTaskExecution",
        "datasync:ListTasks",
        "datasync:StartTaskExecution",
        "s3:GetBucketLocation",
        "s3:ListAllMyBuckets"
      ],
      "Resource": "*"
    },
    {
      "Sid": "IAMPermissions",
      "Effect": "Allow",
      "Action": ["iam:CreateRole", "iam:ListRoles", "iam:CreatePolicy"],
      "Resource": "arn:aws:iam::<ACCOUNT_A_ID>:role/DataSync-*"
    },
    {
      "Sid": "IAMAttachRolePermissions",
      "Effect": "Allow",
      "Action": ["iam:AttachRolePolicy"],
      "Resource": "arn:aws:iam::<ACCOUNT_A_ID>:role/DataSync-*",
      "Condition": {
        "ArnLike": {
          "iam:PolicyARN": [
            "arn:aws:iam::<ACCOUNT_A_ID>:policy/DataSync-*",
            "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess",
            "arn:aws:iam::aws:policy/service-role/AWSDataSyncFullAccess"
          ]
        }
      }
    },
    {
      "Sid": "AllowPassRoleToDataSync",
      "Effect": "Allow",
      "Action": ["iam:PassRole"],
      "Resource": "*",
      "Condition": {
        "StringEquals": { "iam:PassedToService": ["datasync.amazonaws.com"] }
      }
    }
  ]
}
```

> This `iam:PassRole` condition is the one that trips people up — without it, task/location creation fails with a PassRole denial even if the DataSync role itself is set up correctly, because you (the console user) are the one "passing" the role to the DataSync service.

### Destination account (Account B) — user permissions

Just needs S3 permissions to edit the bucket policy and Object Ownership settings on `dest-bucket-b`.

---

## Step 1 — Account A: Create the DataSync IAM Role (Console)

This role is what DataSync assumes to write into Account B's bucket. Since this is cross-account, you must create it **manually** — DataSync's auto-generate-role option in the location wizard only works within the same account.

1. Console (Account A) → **IAM** → **Roles** → **Create role**
2. Trusted entity type: **AWS service**
3. Use case: choose **DataSync** from the dropdown → **Next**
4. On **Add permissions**, choose **Next** (leave blank — you'll add a scoped inline policy after creation)
5. Name it: `source-datasync-role` → **Create role**
6. Open the role → **Permissions** tab → **Add permissions** → **Create inline policy** → **JSON** tab, paste:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Action": [
           "s3:GetBucketLocation",
           "s3:ListBucket",
           "s3:ListBucketMultipartUploads"
         ],
         "Effect": "Allow",
         "Resource": "arn:aws:s3:::dest-bucket-b",
         "Condition": {
           "StringEquals": { "aws:ResourceAccount": "<ACCOUNT_B_ID>" }
         }
       },
       {
         "Action": [
           "s3:AbortMultipartUpload",
           "s3:DeleteObject",
           "s3:GetObject",
           "s3:GetObjectTagging",
           "s3:GetObjectVersion",
           "s3:GetObjectVersionTagging",
           "s3:ListMultipartUploadParts",
           "s3:PutObject",
           "s3:PutObjectTagging"
         ],
         "Effect": "Allow",
         "Resource": "arn:aws:s3:::dest-bucket-b/*",
         "Condition": {
           "StringEquals": { "aws:ResourceAccount": "<ACCOUNT_B_ID>" }
         }
       },
       {
         "Sid": "SourceBucketAccess",
         "Effect": "Allow",
         "Action": ["s3:GetObject", "s3:ListBucket", "s3:GetBucketLocation"],
         "Resource": ["arn:aws:s3:::source-bucket-a", "arn:aws:s3:::source-bucket-a/*"]
       }
     ]
   }
   ```
   Name it `source-datasync-role-inline-policy` → **Create policy**.
7. Copy the role ARN: `arn:aws:iam::<ACCOUNT_A_ID>:role/source-datasync-role`

**What changed from a naive version and why:**
- Full action set includes multipart upload and object-tagging actions (`AbortMultipartUpload`, `ListMultipartUploadParts`, `*ObjectTagging`, `GetObjectVersion*`) — without these, large or versioned-object transfers fail partway through even though basic `PutObject` succeeds on small test files.
- The `aws:ResourceAccount` condition on the destination-bucket statements is a defense-in-depth check confirming the bucket actually belongs to Account B — good practice to keep, and good to be able to explain in an interview.
- Trust policy comes from picking **AWS service → DataSync** in the console wizard, not a hand-pasted custom trust JSON — same end result, but this is what the console actually walks you through, so it's what you'll see live.

---

## Step 2 — Account B: Update the Destination Bucket Policy (Console)

1. Switch console session to **Account B**.
2. **S3** → `dest-bucket-b` → **Permissions** tab → **Bucket policy** → **Edit**
3. Merge in this statement (don't overwrite existing statements if the bucket already has a policy):
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "DataSyncCreateS3LocationAndTaskAccess",
         "Effect": "Allow",
         "Principal": {
           "AWS": "arn:aws:iam::<ACCOUNT_A_ID>:role/source-datasync-role"
         },
         "Action": [
           "s3:GetBucketLocation",
           "s3:ListBucket",
           "s3:ListBucketMultipartUploads",
           "s3:AbortMultipartUpload",
           "s3:DeleteObject",
           "s3:GetObject",
           "s3:ListMultipartUploadParts",
           "s3:PutObject",
           "s3:GetObjectTagging",
           "s3:PutObjectTagging"
         ],
         "Resource": [
           "arn:aws:s3:::dest-bucket-b",
           "arn:aws:s3:::dest-bucket-b/*"
         ]
       }
     ]
   }
   ```
4. **Save changes.**

*(Matches the role's action list exactly — mismatched action lists between the role policy and bucket policy is a common source of partial-transfer failures where small files succeed but multipart/tagged objects fail.)*

---

## Step 3 — Account B: Disable ACLs on the Destination Bucket (Console)

1. Still in Account B → `dest-bucket-b` → **Permissions** tab
2. **Object Ownership** → **Edit**
3. Select **ACLs disabled (recommended)** — this is the exact console label; it's the same underlying setting as "Bucket owner enforced"
4. **Save changes**

This ensures Account B actually owns every object DataSync writes, instead of ownership defaulting to Account A.

---

## Step 4 (Optional) — Account B: KMS Key Policy, if `dest-bucket-b` uses SSE-KMS with a customer-managed key

Skip if the bucket uses SSE-S3 or an AWS-managed key.

1. Account B → **KMS** → the CMK used by `dest-bucket-b`
2. **Key policy** → **Edit** → add:
   ```json
   {
     "Sid": "AllowDataSyncRoleUseOfKey",
     "Effect": "Allow",
     "Principal": { "AWS": "arn:aws:iam::<ACCOUNT_A_ID>:role/source-datasync-role" },
     "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
     "Resource": "*"
   }
   ```
3. **Save changes.**

> For a full walkthrough of SSE-KMS specifics, AWS has a dedicated Storage Blog post on cross-account/cross-region transfers with customer-managed keys — worth a read if this is a real production scenario rather than a lab.

---

## Step 5 — Account A: Create Both DataSync Locations

### 5a. Source location (Console)

1. Switch back to **Account A**.
2. **DataSync** → **Locations** → **Create location**
3. Location type: **Amazon S3** → bucket: `source-bucket-a` → folder `/`
4. IAM role: select `source-datasync-role`
5. **Create location** — note the ARN.

### 5b. Destination location (CloudShell — unavoidable)

The console's location wizard has no field for a role ARN belonging to another account, so it literally cannot complete the cross-account handshake — this isn't a workaround for a missing feature, it's the documented, only path.

1. In **Account A**, open **CloudShell**.
2. Run:
   ```bash
   aws datasync create-location-s3 \
     --s3-bucket-arn arn:aws:s3:::dest-bucket-b \
     --region <dest-bucket-region> \
     --s3-config '{"BucketAccessRoleArn":"arn:aws:iam::<ACCOUNT_A_ID>:role/source-datasync-role"}'
   ```
   Omit `--region` entirely if source and destination buckets are in the same Region.
3. Response returns:
   ```json
   { "LocationArn": "arn:aws:datasync:<region>:<ACCOUNT_A_ID>:location/loc-xxxxxxxxxxxxxxxxx" }
   ```
4. Back in the DataSync console → **Locations** — if the destination bucket is in a different Region, switch the console Region selector to see it listed.

---

## Step 6 — Account A: Create the DataSync Task (Console)

1. **Important — Region rule:** if source and destination buckets are in different Regions, you must create the task itself in the **same Region as the destination location**, or you'll hit a network connection error at run time. Switch the console's top-nav Region selector accordingly before starting.
2. **DataSync** → **Tasks** → **Create task**
3. **Source location**: **Choose an existing location** → select the location from Step 5a
4. **Destination location**: **Choose an existing location** → select the location from Step 5b
5. **Task mode**: choose **Enhanced** — AWS recommends this for cross-account/cross-region transfers specifically because Basic-mode tasks are the ones that hit the network connection error mentioned above.
6. Name the task, point **CloudWatch log group** to a log group in Account A for visibility
7. **Review** → **Create task**

---

## Step 7 — Run and Verify

1. Task details page → **Start** → **Start with defaults**
2. Watch **History** tab: Launching → Preparing → Transferring → Verifying → Success
3. Confirm objects appear in Account B's `dest-bucket-b`
4. Check CloudWatch Logs in Account A for per-object transfer detail

---

## Edge Cases

- **Different Regions**: fully supported, but the task itself must be created in the destination location's Region (Step 6), and Enhanced task mode is the recommended way to avoid connection errors — Basic mode is documented to fail here.
- **Large object counts / small files**: DataSync scales to millions of objects; very small files (sub-1KB) add per-object overhead.
- **Incremental re-runs**: subsequent task runs transfer only changed/new objects by default — useful to demonstrate for an interview.
- **Object Lock on destination**: if enabled, delete-on-sync options in the task will fail — document as a known limitation.
- **Existing bucket policy in Account B**: merge the new `Sid` into the existing `Statement` array — don't replace the whole policy.
- **Versioned objects**: if source or destination buckets have versioning enabled, make sure the role/bucket-policy action lists include `GetObjectVersion`/`GetObjectVersionTagging` (included above) — a policy copied from a non-versioned example will silently fail on versioned objects only.

---

## Troubleshooting Table

| Symptom | Likely Cause | Fix |
|---|---|---|
| `AccessDenied` writing to destination during task run | Bucket policy missing, wrong principal ARN, or action list mismatch between role policy and bucket policy | Re-check Step 2; action lists in role inline policy and bucket policy must match |
| Network/connection error when starting the task | Task created in wrong Region (not destination's Region), or using Basic mode across accounts+Regions | Recreate task in destination's Region, or switch task mode to Enhanced (Step 6) |
| `iam:PassRole` AccessDenied when creating the location or task | Your own console user identity lacks the `iam:PassRole` condition for `datasync.amazonaws.com` | Add the PassRole statement from Prerequisites to your own IAM user/role, not the DataSync role |
| Destination location doesn't appear in the task wizard dropdown | Wrong Region selected in console, or location was created in a different Region than currently viewed | Switch the console Region selector to match the destination bucket's Region |
| `kms:Decrypt` / `AccessDeniedException` on SSE-KMS bucket | Key policy not updated | Complete Step 4 |
| Objects land in destination but Account B shows ownership warnings | ACLs still enabled | Complete Step 3 |
| Small files transfer fine, larger/multipart or versioned objects fail | Action list missing `AbortMultipartUpload`/`ListMultipartUploadParts`/`GetObjectVersion*` | Use the full action list in Steps 1 & 2, not a trimmed-down version |
| Console "Create location" wizard won't let you pick a bucket in another account | Expected — console cannot create cross-account locations at all | Use CloudShell per Step 5b |

---

## Interview Q&A

**Q: Why can't you create a cross-account S3 location entirely in the console?**
A: The location wizard has no field to specify a role ARN belonging to another account, so it can't complete the destination-account trust handshake — AWS's own tutorial documents `create-location-s3` via CLI/CloudShell as the only path for this one step.

**Q: Where do the DataSync task and locations actually live — source or destination account?**
A: Both locations and the task are created in the source account. The destination account creates zero DataSync resources — it only updates its bucket policy (and KMS key policy, if applicable) to trust the source account's role.

**Q: Why do you need both an IAM role policy and a bucket policy granting the same permissions?**
A: S3 cross-account access requires the bucket owner (destination account) to independently trust the exact IAM principal, even if that principal's own policy already allows the action — it's implicit deny until both sides agree.

**Q: What's `iam:PassRole` doing in this setup and why does it trip people up?**
A: It's not about the DataSync role's own permissions — it's about whether *you*, the console user creating the location/task, are allowed to hand that role off to the DataSync service. Missing this on your own user/role causes a PassRole denial even when the DataSync role itself is perfectly configured.

**Q: Why does AWS recommend Enhanced task mode for cross-account transfers?**
A: Basic-mode tasks are documented to throw network connection errors when the source and destination are in different accounts and Regions; Enhanced mode avoids this, or alternatively you can keep the task in the same Region as the destination location.

**Q: How would you extend this for ongoing replication instead of one-time migration?**
A: Attach a schedule to the task, or compare against native S3 Cross-Account Replication — DataSync gives verification, filtering, and centralized logging per run; native replication is event-driven and simpler but less controllable per execution.

---

## Cheat Sheet

```
Role: source-datasync-role (Account A) — trust = AWS service: DataSync
Role permissions: full S3 action set on dest bucket (incl. multipart + tagging + versioning)
                  + read on source bucket
                  + aws:ResourceAccount condition recommended
Your own console user: needs iam:PassRole condition for datasync.amazonaws.com
Destination bucket policy: same action list, principal = exact source-datasync-role ARN
Destination bucket: Object Ownership = ACLs disabled
KMS (if SSE-KMS): key policy grants role kms:Decrypt + kms:GenerateDataKey
Task Region rule: create task in the DESTINATION location's Region
Task mode: Enhanced (avoids cross-account/cross-Region connection errors)
Console: everything EXCEPT destination location creation (CloudShell only)
```

---

## Mastery Checklist

- [ ] Can explain why only the destination *location* requires CLI, not the task, source location, or role
- [ ] Can state both permission layers required (IAM role policy + bucket policy) and why action lists must match
- [ ] Can explain `iam:PassRole` in this context, distinct from the DataSync role's own permissions
- [ ] Can explain the Region rule for task creation and why Enhanced mode avoids it
- [ ] Can identify KMS as a silent failure point even after S3 permissions are correct
- [ ] Successfully ran a real cross-account transfer end-to-end with CloudWatch logs to prove it

---

## Cleanup (dependency order)

1. **Account A** — Stop/delete the DataSync **task**
2. **Account A** — Delete both DataSync **locations** (source, then destination)
3. **Account A** — Delete the inline policy, then delete `source-datasync-role`
4. **Account B** — Remove the added statement from the bucket policy (keep the rest if other rules exist)
5. **Account B** — Revert KMS key policy change, if made
6. **Account B** — Empty and delete `dest-bucket-b` if lab-only
7. **Account A** — Empty and delete `source-bucket-a` if lab-only
8. **Account A** — Delete the CloudWatch Logs group used for task logging
