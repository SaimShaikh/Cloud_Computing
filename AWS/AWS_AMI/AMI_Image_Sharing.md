# AWS Lab: Configure Image Sharing (Amazon Machine Image - AMI)

## Objective

Learn how to create an Amazon Machine Image (AMI), share it with another AWS account, and launch an EC2 instance from the shared image.

---

# Lab Architecture

```text
                AWS Account A (Source)
+------------------------------------------------+
| EC2 Instance                                   |
|        │                                       |
|        ▼                                       |
|   Create AMI                                   |
|        │                                       |
|        ▼                                       |
|   Amazon Machine Image (AMI)                   |
|        │                                       |
| Edit Launch Permissions                        |
|        │                                       |
| Share with AWS Account B                       |
+------------------------------------------------+
                    │
                    ▼
                AWS Account B (Target)
+------------------------------------------------+
| Shared AMI                                     |
|        │                                       |
|   (Recommended) Copy AMI                       |
|        │                                       |
|        ▼                                       |
| Launch EC2 Instance                            |
+------------------------------------------------+
```

---

# Prerequisites

- Two AWS accounts (Source and Target)
- Existing EC2 instance in the source account
- IAM permissions for EC2 and AMI operations
- If using encrypted EBS volumes, permissions to manage the KMS key

---

# Step 1: Create an AMI

1. Sign in to the **Source AWS Account**.
2. Open **EC2 Console**.
3. Select your EC2 instance.
4. Choose:

```
Actions
 └── Image and templates
      └── Create image
```

5. Enter:
   - **Image name:** `golden-web-v1`
   - **Description:** Golden image for lab
   - Optionally enable **No reboot**
6. Click **Create Image**.
7. Wait until the AMI state changes to **Available**.

---

# Step 2: Verify the AMI

Go to:

```
EC2 → AMIs
```

Verify:
- AMI State = Available
- Note the AMI ID (e.g. `ami-0abc123456789def0`)

---

# Step 3: Share the AMI

1. Select the AMI.
2. Click:

```
Actions
 └── Edit AMI Permissions
```

3. Keep the AMI **Private**.
4. Under **AWS Account IDs**, add the 12-digit target account ID.
5. Click **Save**.

---

# Step 4: Share the KMS Key (Encrypted AMIs Only)

If the AMI uses encrypted EBS snapshots:

1. Open **AWS KMS**.
2. Select the customer-managed key.
3. Edit the key policy or grants.
4. Allow the target AWS account to use the key.

> If you skip this step, the target account will not be able to launch instances from the encrypted AMI.

---

# Step 5: View the Shared AMI (Target Account)

1. Sign in to the **Target AWS Account**.
2. Open **EC2 Console**.
3. Go to:

```
EC2 → AMIs
```

4. Filter by **Shared with me**.
5. Verify that the shared AMI is listed.

---

# Step 6: Copy the Shared AMI (Recommended)

1. Select the shared AMI.
2. Choose:

```
Actions
 └── Copy AMI
```

3. Give it a name such as:

```
golden-web-v1-copy
```

4. Wait until the copied AMI becomes **Available**.

---

# Step 7: Launch an EC2 Instance

1. Select the copied AMI.
2. Click **Launch Instance**.
3. Choose:
   - Instance type
   - VPC and subnet
   - Security group
   - Key pair
4. Launch the instance.
5. Connect using SSH or EC2 Instance Connect and verify it matches the original configuration.

---

# AWS CLI Commands

## Share an AMI

```bash
aws ec2 modify-image-attribute \
  --image-id ami-0abc123456789def0 \
  --launch-permission "Add=[{UserId=222222222222}]"
```

## Check Launch Permissions

```bash
aws ec2 describe-image-attribute \
  --image-id ami-0abc123456789def0 \
  --attribute launchPermission
```

## Copy the Shared AMI

```bash
aws ec2 copy-image \
  --source-region ap-south-1 \
  --source-image-id ami-0abc123456789def0 \
  --name "golden-web-v1-copy"
```

---

# Validation Checklist

- [ ] AMI created successfully
- [ ] AMI state is **Available**
- [ ] Launch permissions include the target account
- [ ] KMS key shared (if encrypted)
- [ ] Target account can see the AMI
- [ ] AMI copied successfully
- [ ] EC2 instance launched from the copied AMI
- [ ] Application and configuration verified

---

# Common Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| Shared AMI not visible | Wrong region | Switch to the correct AWS region |
| Launch failed | Missing KMS permissions | Share the KMS key |
| Copy failed | Missing IAM permissions | Grant EC2 image permissions |
| AMI Pending | Snapshot still being created | Wait until the AMI is Available |

---

# Best Practices

- Keep AMIs private unless public distribution is required.
- Copy shared AMIs into your own account before production use.
- Use versioned names (e.g., `golden-web-v1.0.0`).
- Remove unused AMIs and snapshots to reduce costs.
- Periodically audit launch permissions and KMS access.

---

# Expected Outcome

By the end of this lab you will be able to:

- Create an AMI from an EC2 instance.
- Share the AMI with another AWS account.
- Handle encrypted AMI sharing using KMS.
- Copy a shared AMI into another account.
- Launch a new EC2 instance from the shared AMI.
