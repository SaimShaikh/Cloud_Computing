# AWS AMI Cross-Account Image Sharing — End-to-End Lab Guide

## Scenario

Your team builds a "golden" AMI in one AWS account (Source) and needs to distribute it to another account (Target) — a common pattern for multi-account setups (dev/prod split, shared services account, or a customer/vendor handoff). You need to prove the full lifecycle: build the AMI, share it safely, copy it into the target account so it's no longer dependent on the source, launch from it, and understand exactly what breaks (and what doesn't) if the source account deregisters the original.

This lab uses **two AWS accounts** in the **same Region** and covers both plain and KMS-encrypted AMIs.

---

## Architecture

```
                 Source AWS Account (111111111111)
+--------------------------------------------------------------+
| EC2 Instance (ami-source-server)                              |
|      │                                                        |
|      ▼                                                        |
|  Create Image (no reboot decision)                            |
|      │                                                        |
|  AMI (state: pending → available)                             |
|      │                                                        |
|  Backing EBS Snapshot(s)                                       |
|      │                                                        |
|  Edit AMI Launch Permissions ── Add Target Account ID          |
|      │                                                        |
|  (If encrypted) Share KMS CMK key policy with Target            |
|      │                                                        |
+------|---------------------------------------------------------+
       │  Share
       ▼
                 Target AWS Account (222222222222)
+--------------------------------------------------------------+
| EC2 → AMIs → Owner = Shared with me                            |
|      │                                                        |
|      ▼                                                        |
|  Copy AMI  (recommended — breaks dependency on Source)         |
|      │                                                        |
|  New AMI + New Snapshot(s) — Owner = Owned by me                |
|      │                                                        |
|  Launch EC2 (ami-target-server)                                |
|      │                                                        |
|  Verify data / app parity with source                          |
+--------------------------------------------------------------+

Failure scenario:
Source deregisters original AMI
        │
        ├── Target did NOT copy  → shared AMI reference disappears, cannot launch
        └── Target DID copy      → copied AMI + snapshot are fully independent, unaffected
```

---

## Prerequisites

| Item | Value (example) |
|---|---|
| Source Account ID | 111111111111 |
| Target Account ID | 222222222222 |
| Region (both accounts) | ap-south-1 (Mumbai) |
| Source instance | ami-source-server (t3.micro) |
| AMI name | golden-web-v1 |
| Encryption | Optional — CMK example: `alias/ami-share-key` |
| IAM permission (source) | `ec2:CreateImage`, `ec2:ModifyImageAttribute`, `ec2:DescribeImages` |
| IAM permission (target) | `ec2:CopyImage`, `ec2:DescribeImages`, `ec2:RunInstances`, `kms:CreateGrant` (if encrypted) |

> AMIs are **Regional resources** — sharing only works within the same Region unless you explicitly copy across Regions (Part 7).

---

## Part 1 — Prepare the Source Environment

### Phase 1 – Launch the Source EC2 Instance

1. **EC2 → Instances → Launch Instance**.
2. Name: `ami-source-server`.
3. AMI: Amazon Linux 2023. Instance type: `t3.micro`.
4. Key pair: select/create one.
5. Network: default VPC, public subnet, enable **Auto-assign Public IP**.
6. Security group: allow SSH (22) from your IP.
7. Storage: 8–20 GiB gp3.
8. Launch.

### Phase 2 – Install Test Application / Data

Connect via SSH, then create something verifiable so you can prove the copied AMI is a true match later:

```bash
sudo yum install -y httpd
sudo systemctl enable --now httpd
echo "<h1>Golden AMI v1 - $(hostname)</h1>" | sudo tee /var/www/html/index.html
mkdir ~/ami-test
echo "AMI Source Marker v1" > ~/ami-test/marker.txt
```

Verify:
```bash
curl localhost
cat ~/ami-test/marker.txt
```

### Phase 3 – Decide: Reboot or No-Reboot AMI Creation

| Option | Behavior | When to use |
|---|---|---|
| Default (reboot) | AWS stops the instance briefly to flush filesystem buffers before snapshotting → guarantees a **crash-consistent to application-consistent** image | Standard production golden images |
| No reboot | Instance stays running; snapshot is taken live | Only when downtime is unacceptable **and** you accept a higher risk of an inconsistent filesystem state (e.g., open DB writes not flushed) |

For this lab we'll use the default (reboot) for a clean, consistent AMI.

---

## Part 2 — Create the AMI

### Phase 4 – Create the Image

1. **EC2 → Instances → select `ami-source-server`**.
2. **Actions → Image and templates → Create image**.
3. Image name: `golden-web-v1`.
4. Description: `Golden web AMI - Apache preinstalled - v1`.
5. Instance volumes: confirm root volume is included; add tags if desired (`Name=golden-web-v1`, `Environment=POC`).
6. Leave **No reboot** unchecked (default) for this lab.
7. Click **Create Image**.

### Phase 5 – Monitor AMI Creation

**EC2 → AMIs → Owned by me**. Status flow:

```
pending → available
```

This can take a few minutes — AWS is creating the underlying EBS snapshot(s) in the background.

### Phase 6 – Verify the AMI and Its Snapshot

1. Confirm **State = Available**. Note the **AMI ID** (`ami-xxxxxxxxxxxxxxxxx`).
2. **EC2 → Snapshots → Owned by me** — confirm a new snapshot exists, description referencing `golden-web-v1`, and its **State = Completed**.
3. Click the AMI → **Storage tab** — confirm the snapshot ID matches what you see under Snapshots.

---

## Part 3 — Share the AMI Cross-Account

### Phase 7 – Get the Target Account ID

In the **Target** account: top-right account menu → copy the 12-digit **Account ID** (e.g., `222222222222`). You'll need this in the Source account next.

### Phase 8 – Edit AMI Launch Permissions

In the **Source** account:

1. **EC2 → AMIs** → select `golden-web-v1`.
2. **Actions → Edit AMI permissions** (previously called "Edit AMI Permissions").
3. Under **AMI permissions**, keep visibility as **Private**.
4. Click **Add account** → enter the Target's 12-digit Account ID.
5. Click **Save changes**.

> Avoid "Public" — that shares the AMI with every AWS account in the world. Only use it for a genuinely public marketplace-style image.

### Phase 9 – Share the KMS Key (Encrypted AMIs Only)

If the AMI's backing snapshot is encrypted with a **customer-managed key (CMK)**:

1. **AWS KMS → Customer managed keys** → select the CMK (e.g., `alias/ami-share-key`).
2. **Key policy → Edit** (or use **Key users** section if using the default policy view).
3. Add the Target account/role as a key user, or add a statement granting:
   - `kms:Decrypt`
   - `kms:DescribeKey`
   - `kms:CreateGrant`
   - `kms:GenerateDataKeyWithoutPlaintext`
4. Save the policy.

**Without this step, the target account can see the shared AMI but launch and copy will fail with an access-denied / KMS error.** AWS-managed default EBS keys (`aws/ebs`) **cannot** be shared across accounts — you must use a CMK for cross-account encrypted AMI sharing.

### Phase 10 – (Optional) Share at Scale via AWS Organizations / RAM

For sharing with many accounts instead of one at a time:

1. **EC2 → AMIs → Edit AMI permissions** → under **Shared organizations/OUs**, add your AWS Organization ID or a specific OU ID (requires AWS Organizations with the "share with organization" feature enabled).
2. Alternative: use **AWS Resource Access Manager (RAM)** to share the AMI resource with an Organizational Unit or specific accounts centrally.

This avoids manually adding account IDs one by one as your account count grows.

---

## Part 4 — Access and Copy the AMI (Target Account)

### Phase 11 – Switch to the Correct Region

Sign in to the **Target** account and switch to the **same Region** the Source used (e.g., `ap-south-1`). AMI shares do not appear in any other Region.

### Phase 12 – View the Shared AMI

1. **EC2 → Images → AMIs**.
2. Change the **Owner** filter dropdown to **Shared with me**.
3. Confirm `golden-web-v1` appears, with the Source account ID listed as Owner.

### Phase 13 – Copy the Shared AMI (Recommended)

1. Select the shared AMI → **Actions → Copy AMI**.
2. Destination Region: same Region (or a different one — see Part 7).
3. Name: `golden-web-v1-target-copy`.
4. Encryption: 
   - If the source AMI was unencrypted, you can optionally encrypt the copy with a target-account KMS key.
   - If the source AMI was encrypted, you can re-encrypt with a **key you own** in the target account — this fully removes the KMS dependency on the source account too.
5. Click **Copy AMI**.

### Phase 14 – Monitor and Verify the Copy

1. **EC2 → AMIs → Owned by me** in the target account.
2. Status flow: `pending → available`.
3. Once available, confirm:
   - **Owner = Owned by me** (target account ID, not source).
   - A new, independent snapshot now exists under **Snapshots → Owned by me** in the target account.

This is the key moment — the target account now has its **own** copy of the AMI and its snapshot, fully decoupled from the source.

---

## Part 5 — Launch and Validate

### Phase 15 – Launch an EC2 Instance from the Copied AMI

1. **EC2 → AMIs → select `golden-web-v1-target-copy`**.
2. **Launch instance from AMI**.
3. Name: `ami-target-server`.
4. Instance type: `t3.micro`. Key pair: target account's own key pair.
5. Network: target account's VPC/subnet, enable Auto-assign Public IP.
6. Security group: allow SSH (22) and HTTP (80).
7. Launch.

### Phase 16 – Verify Application and Data Parity

Connect via SSH:

```bash
curl localhost
cat ~/ami-test/marker.txt
```

Expected output matches the source exactly:
```
<h1>Golden AMI v1 - ip-xxx-xxx-xxx-xxx</h1>
AMI Source Marker v1
```

### Phase 17 – Confirm Configuration Match

Compare key attributes between source and target instance (OS version, installed packages, root volume size) to confirm the AMI captured the full expected state:

```bash
cat /etc/os-release
rpm -qa | grep httpd
df -h /
```

---

## Part 6 — Deregistration and Failure Scenarios

### Phase 18 – Deregister the Source AMI (Test)

In the **Source** account:

1. **EC2 → AMIs** → select `golden-web-v1`.
2. **Actions → Deregister AMI** → confirm.

> Deregistering an AMI does **not** automatically delete its backing EBS snapshot(s).

### Phase 19 – Delete the Orphaned Source Snapshot

1. **EC2 → Snapshots** → select the snapshot(s) tied to the deregistered AMI.
2. **Actions → Delete snapshot** → confirm.

### Phase 20 – Validate Scenario 1: Target Did NOT Copy

If a *different* target account only viewed the shared AMI but never copied it:

```
Source deregisters AMI
        │
        ▼
Shared AMI disappears from "Shared with me" in target
        │
        ▼
Target cannot launch new instances from it
```

Confirm in that target account: **EC2 → AMIs → Shared with me** — the AMI is gone. This is expected AWS behavior, not a bug.

### Phase 21 – Validate Scenario 2: Target DID Copy

In your lab's target account (which copied the AMI in Phase 13):

1. **EC2 → AMIs → Owned by me** → confirm `golden-web-v1-target-copy` is still **Available**, even though the source original was deregistered in Phase 18.
2. Launch one more test instance from it to confirm it still works.

```
Source deregisters original AMI
        │
        ▼
Copied AMI in target account is fully independent
        │
        ▼
Target continues launching EC2 with zero impact
```

This proves the copy step is what actually protects the target account from upstream changes.

---

## Part 7 — Production Hardening

### Phase 22 – Restrict Deregistration Permissions (IAM)

Limit who can deregister production AMIs with an IAM policy statement such as:

```json
{
  "Effect": "Deny",
  "Action": "ec2:DeregisterImage",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:PrincipalTag/Role": "AMIAdmin"
    }
  }
}
```

Attach as a guardrail (e.g., via SCP at the OU level, or a permissions boundary) so only designated admins can deregister golden images.

### Phase 23 – Protect Against Accidental Deletion with Recycle Bin

1. **EC2 → Recycle Bin → Create retention rule**.
2. Resource type: **AMIs** (and/or **Snapshots**).
3. Retention period: e.g., 7 days.
4. Tag-based scope (optional): only apply to resources tagged `Environment=Production`.

Deregistered AMIs matching the rule go to the Recycle Bin instead of being permanently removed immediately, giving you a recovery window.

### Phase 24 – Deprecate Instead of Deregister (Soft EOL)

For golden images being phased out but still needed for a transition period:

1. **EC2 → AMIs → select the AMI → Actions → Edit deprecation settings**.
2. Set a **Deprecation date**.

A deprecated AMI still works and can still be launched/copied, but is flagged and excluded from default `describe-images` results after the date — a safer way to signal "don't use this for new builds" without breaking anything already depending on it.

### Phase 25 – Cross-Region Distribution

Since AMIs are Regional, if the target account operates in a different Region, copy across Regions instead of (or in addition to) across accounts:

1. Shared AMI still needs to be viewed in the **same Region as the source** first (Phase 12).
2. During **Copy AMI** (Phase 13), set **Destination Region** to the target Region (e.g., copy from `ap-south-1` to `ap-southeast-1`).
3. Repeat launch validation (Phase 15–17) in the new Region.

### Phase 26 – Automate Distribution at Scale

For recurring golden-image pipelines instead of manual sharing:

- **AWS Image Builder** — define a pipeline that builds, tests, and automatically distributes AMIs to multiple accounts/Regions on a schedule.
- **EventBridge + Lambda** — trigger auto-copy into target accounts whenever a new AMI is created and tagged `share=true`.
- **Central Image Account pattern** — one dedicated account owns and publishes all golden AMIs; other accounts only ever consume copies, never depend on ad hoc peer-to-peer sharing.

---

## AWS CLI Reference

**Create an image:**
```bash
aws ec2 create-image \
  --instance-id i-xxxxxxxxxxxxxxxxx \
  --name "golden-web-v1" \
  --description "Golden web AMI - Apache preinstalled - v1"
```

**Share an AMI with another account:**
```bash
aws ec2 modify-image-attribute \
  --image-id ami-xxxxxxxxxxxxxxxxx \
  --launch-permission "Add=[{UserId=222222222222}]"
```

**Verify launch permissions:**
```bash
aws ec2 describe-image-attribute \
  --image-id ami-xxxxxxxxxxxxxxxxx \
  --attribute launchPermission
```

**Copy a shared AMI (run in target account):**
```bash
aws ec2 copy-image \
  --source-region ap-south-1 \
  --source-image-id ami-xxxxxxxxxxxxxxxxx \
  --name "golden-web-v1-target-copy"
```

**Deregister an AMI:**
```bash
aws ec2 deregister-image --image-id ami-xxxxxxxxxxxxxxxxx
```

**Delete an orphaned snapshot:**
```bash
aws ec2 delete-snapshot --snapshot-id snap-xxxxxxxxxxxxxxxxx
```

**Deprecate an AMI:**
```bash
aws ec2 enable-image-deprecation \
  --image-id ami-xxxxxxxxxxxxxxxxx \
  --deprecate-at "2027-01-01T00:00:00"
```

---

## Edge Cases & Gotchas

| Edge Case | Behavior / Fix |
|---|---|
| Target checks the wrong Region | Shared AMI simply won't appear — no error message. Always confirm both accounts are in the identical Region. |
| AMI backed by AWS-managed default key (`aws/ebs`) | Cannot be shared cross-account at all — the CMK requirement is not optional for encrypted cross-account sharing. |
| No-reboot AMI of a database instance | Risk of an application-inconsistent snapshot (in-flight writes not flushed) — avoid for stateful data-heavy workloads. |
| Target copies AMI, then Source updates KMS key policy | Already-copied AMI is unaffected if re-encrypted with the target's own key during copy; if not re-encrypted, it still depends on the original CMK. |
| Multiple EBS volumes on the source instance | `Create Image` snapshots **all** attached volumes by default — verify only the intended volumes are included before sharing. |
| AMI shared but launch permission later revoked | Any instance **already launched** from the AMI keeps running; only *new* launches from that AMI are blocked going forward. |
| Deregistering while an Auto Scaling Group still references the AMI | ASG launch template becomes invalid for new scale-out events — update the launch template to a valid AMI first. |

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Shared AMI not visible in target | Wrong Region selected | Switch to the exact Region the source used |
| Shared AMI not visible in target | Account ID entered incorrectly in Edit AMI permissions | Re-check the 12-digit ID, re-add, re-save |
| Copy/Launch fails with encryption/access error | KMS key not shared with target | Update CMK key policy to grant target account decrypt/grant permissions |
| Copy fails with permission error | Target IAM user/role missing `ec2:CopyImage` | Attach an IAM policy granting `ec2:CopyImage` and related EC2 permissions |
| Instance launched from copy boots but app/data missing | AMI created with No-reboot on a system with unflushed writes | Recreate the AMI with the default reboot option |
| Shared AMI suddenly gone from "Shared with me" | Source deregistered the original | Expected behavior — always copy immediately after confirming the share |
| Snapshots pile up in source account | Deregistering an AMI doesn't delete snapshots | Manually delete the associated snapshot(s) after deregistering |
| Cannot deregister a production AMI | IAM guardrail/SCP restricting the action | Confirm with the AMI admin/owner before removing the restriction |



