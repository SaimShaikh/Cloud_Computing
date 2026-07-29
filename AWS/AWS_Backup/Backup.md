
# AWS Backup, EBS Snapshots & Disaster Recovery 

## Scenario

You are the DevOps/Cloud engineer for a production workload running on a single EC2 instance. Leadership wants proof that:

1. The instance and its data can be backed up **automatically** on a schedule (AWS Backup).
2. A **manual, point-in-time** snapshot mechanism exists for ad-hoc protection before risky changes (EBS Snapshots).
3. The organization has a **documented recovery posture** — retention, cross-Region DR, immutability, audit, and measured RPO/RTO (Recovery Policies).

This lab builds all three end-to-end: create the source EC2 → configure AWS Backup → force a backup → restore it → take a manual EBS snapshot → restore only the volume → layer on production-grade recovery policies (lifecycle, cross-Region copy, Vault Lock, Audit Manager, RPO/RTO). Everything is done console-first so each click maps to a concept you can explain in an interview.

---

## Architecture

<img width="1145" height="1374" alt="image" src="https://github.com/user-attachments/assets/43916cc7-df98-4186-b45d-4c72a28babb1" />


---

## Prerequisites

| Item | Value (example) |
|---|---|
| Region | ap-south-1 (Mumbai) |
| DR Region | ap-southeast-1 (Singapore) |
| Instance name | backup-demo-server |
| Instance type | t3.micro |
| Root volume | 20 GiB, gp3 |
| Security group | backup-sg |
| Key pair | backup-key |
| Test data path | ~/backup-demo/ |

---

## Part 1 — Set Up the Source Environment

### Phase 1 – Launch the Test EC2 Instance

1. Go to **EC2 → Instances → Launch Instance**.
2. Name: `backup-demo-server`.
3. AMI: Amazon Linux 2023.
4. Instance type: `t3.micro`.
5. Key pair: select or create `backup-key`.
6. Network: default VPC, public subnet, **Auto-assign Public IP: Enable**.
7. Security group: create/select `backup-sg` (allow SSH from your IP).
8. Storage: 20 GiB, gp3.
9. Click **Launch Instance**.

### Phase 2 – Create Test Data

Connect via SSH or EC2 Instance Connect, then:

```bash
mkdir ~/backup-demo
echo "This is Backup POC Version 1" > ~/backup-demo/data.txt
fallocate -l 100M ~/backup-demo/test.img
ls -lh ~/backup-demo
```

Expected:
```
data.txt
test.img
```

---

## Part 2 — Configure AWS Backup

### Phase 3 – Create a Backup Vault

1. Go to **AWS Backup → Backup vaults → Create Backup Vault**.
2. Name: `Production-Backup-Vault`.
3. Encryption key: **AWS managed key** (default).
4. Click **Create Backup Vault**.

### Phase 4 – Create a Backup Plan

1. Go to **AWS Backup → Backup plans → Create Backup Plan**.
2. Choose **Build a new plan**.
3. Plan name: `Daily-EC2-Backup`.
4. Rule name: `DailyBackupRule`.
5. Backup vault: `Production-Backup-Vault`.
6. Schedule: Daily.
7. Start window: 8 hours; Completion window: 7 days.
8. Lifecycle: **Delete after 30 days** (transition to cold storage not supported for EC2/EBS).
9. Click **Create Plan**.

### Phase 5 – Assign Resources

1. Inside `Daily-EC2-Backup`, go to **Resource assignments → Assign resources**.
2. Assignment name: `EC2-Assignment`.
3. IAM role: **Default** (AWSBackupDefaultServiceRole).
4. Resource selection: choose **Resource type = EC2**, select `backup-demo-server`.
   - Alternative: assign by **tag** (e.g., `Environment = POC`) for scale.
5. Click **Assign resources**.

### Phase 6 – Trigger an On-Demand Backup

1. Go to **AWS Backup → Backup vaults → Production-Backup-Vault**.
2. Click **Create on-demand backup**.
3. Resource type: EC2 → select `backup-demo-server`.
4. IAM role: Default.
5. Retention: 30 days.
6. Click **Create on-demand backup**.

### Phase 7 – Monitor the Backup Job

Go to **Jobs → Backup jobs**. Watch status transition:

```
CREATED → RUNNING → COMPLETED
```

Wait for **COMPLETED** before proceeding.

### Phase 8 – Confirm the Recovery Point

1. Go to **Backup vaults → Production-Backup-Vault**.
2. Confirm a recovery point exists with:
   - Resource Type: EC2
   - Status: Completed
   - Creation Time
   - Expiration Date (per retention rule)

---

## Part 3 — Restore the EC2 Instance from AWS Backup

> Do not terminate the original instance — restore to a **new** instance so you can compare side by side.

### Phase 9 – Start the Restore

1. Open the recovery point → click **Restore**.
2. Choose **Restore entire EC2 instance** → **Next**.

### Phase 10 – Configure the Restored Instance

| Setting | Value |
|---|---|
| Restore role | Default |
| AMI | Auto-filled |
| Instance type | t3.micro |
| Root volume | gp3 |
| Availability Zone | Same as original |

Leave defaults unchanged unless testing a different configuration.

### Phase 11 – Name, Network, Security, Key, IAM

1. Instance name: `backup-demo-server-restored`.
2. Networking: same VPC and subnet as original; enable **Auto Assign Public IP** if connecting directly.
3. Security group: `backup-sg` (same as original).
4. Key pair: same as original (`backup-key`).
5. IAM role: same instance profile as original, if any (else leave empty).
6. Review → click **Restore backup**.

### Phase 12 – Monitor the Restore Job

**Restore jobs** status flow:

```
CREATED → RUNNING → COMPLETED
```

### Phase 13 – Verify the Restored Instance

1. **EC2 → Instances** — you should now see both `backup-demo-server` and `backup-demo-server-restored`.
2. Wait for **Running** + **2/2 status checks passed**.
3. Connect via SSH/EC2 Instance Connect:
   ```bash
   hostname
   ls -l ~/backup-demo
   cat ~/backup-demo/data.txt
   ```
   Expected file list: `data.txt`, `test.img`. Expected content: `This is Backup POC Version 1`.

### Phase 14 – Compare Original vs Restored

If you had since modified the original file (e.g., appended "Version 2"), the original will show the newer content while the restored instance shows the state **as of the recovery point**:

| Instance | data.txt content |
|---|---|
| Original | This is Backup POC Version 1 / Version 2 |
| Restored | This is Backup POC Version 1 |

This proves AWS Backup restored the earlier captured state.

### Phase 15 – Measure RTO

| Event | Time |
|---|---|
| Restore Started | 11:10 |
| Instance Running | 11:18 |

**Measured RTO = 8 minutes**

### Phase 16 – Verify the Restored EBS Volume

**EC2 → Volumes** — confirm the new volume:
- State: `in-use`
- Size matches original (e.g., 20 GiB)
- Type: gp3

### Phase 17 – Functional Test

```bash
echo "Restore Successful" > ~/backup-demo/restore-test.txt
cat ~/backup-demo/restore-test.txt
```
Expected: `Restore Successful` — confirms the restored volume is writable.

### Phase 18 – Document AWS Backup Results

| Test | Result |
|---|---|
| Backup Completed | ✅ |
| Recovery Point Created | ✅ |
| Restore Job Completed | ✅ |
| EC2 Booted Successfully | ✅ |
| Data Restored Correctly | ✅ |
| Application Accessible | ✅ |
| RTO Measured | ✅ |

**Optional cleanup:** stop/terminate `backup-demo-server-restored` and delete its EBS volume if you're done comparing. Keep the original instance for Part 4.

---

## Part 4 — Manual EBS Snapshot POC

| AWS Backup | Manual EBS Snapshot |
|---|---|
| Automated | Manual |
| Backup plans | Individual snapshots |
| Recovery points | Snapshots |
| Lifecycle policies | Manual retention |
| Cross-service backups | EBS volumes only |

**Goal:** prove a manual snapshot restores the volume's exact state at capture time, including files later deleted.

### Phase 19 – Identify the EBS Volume

1. **EC2 → Volumes** → find the root volume of `backup-demo-server`.
2. Confirm **Attached Instance = backup-demo-server**.

### Phase 20 – Create the Manual Snapshot

1. Select the volume → **Actions → Create Snapshot**.
2. Description: `Manual Snapshot Before Data Change`.
3. Tags: `Name = manual-ebs-snapshot`, `Environment = POC`.
4. Click **Create Snapshot**.

### Phase 21 – Monitor the Snapshot

**Snapshots** page: status flows `Pending → Completed`. Wait for **Completed**.

### Phase 22 – Modify Live Data (Post-Snapshot Changes)

SSH into the original EC2:

```bash
cat ~/backup-demo/data.txt
# This is Backup POC Version 1 / Version 2

echo "Version 3" >> ~/backup-demo/data.txt
cat ~/backup-demo/data.txt
# This is Backup POC Version 1 / Version 2 / Version 3

rm ~/backup-demo/test.img
ls -lh ~/backup-demo
# only data.txt remains
```

Live EC2 has now diverged from the snapshot.

### Phase 23 – Restore Snapshot to a New Volume

1. **EC2 → Snapshots** → select `Manual Snapshot Before Data Change`.
2. **Actions → Create Volume**.
3. Availability Zone: same AZ as `backup-demo-server` (e.g., `ap-south-1a`).
4. Volume type: gp3. Size: default (matches snapshot).
5. Click **Create Volume**.

### Phase 24 – Verify the New Volume

**EC2 → Volumes**: original shows `in-use`, new volume shows `available`.

### Phase 25 – Attach the Restored Volume

1. Select the new volume → **Actions → Attach Volume**.
2. Instance: `backup-demo-server`.
3. Device name: `/dev/sdf`.
4. Click **Attach**.

### Phase 26 – Identify the Device in Linux

```bash
lsblk
```
Expected pattern:
```
xvda    20G
└─xvda1
xvdf    20G
```

On Nitro instances, use `sudo fdisk -l` — the device may appear as `/dev/nvme1n1` instead of `/dev/xvdf`.

### Phase 27 – Mount the Restored Volume

```bash
sudo mkdir /restore-volume
```

If the device is `/dev/nvme1n1p1`:
```bash
sudo mount /dev/nvme1n1p1 /restore-volume
```
If the device is `/dev/xvdf1`:
```bash
sudo mount /dev/xvdf1 /restore-volume
```

> Mount the **partition** (e.g., `p1` or `1`), not the raw disk — the root EBS volume contains a partition table.

### Phase 28 – Verify Restored Data

```bash
ls -la /restore-volume/home/ec2-user/backup-demo
cat /restore-volume/home/ec2-user/backup-demo/data.txt
```

Expected: `This is Backup POC Version 1 / Version 2` — **no "Version 3"**.

```bash
ls -lh /restore-volume/home/ec2-user/backup-demo
```
Expected: both `data.txt` and `test.img` present, even though `test.img` was deleted on the live instance.

### Phase 29 – Compare Live vs Snapshot

| | Live EC2 | Snapshot (restored) |
|---|---|---|
| data.txt | Version 1 / 2 / 3 | Version 1 / 2 |
| test.img | ❌ deleted | ✅ present |

This demonstrates true point-in-time recovery.

### Phase 30 – Clean Up the Manual Snapshot POC

```bash
sudo umount /restore-volume
```
Then in the console: **EC2 → Volumes** → select restored volume → **Actions → Detach Volume** → delete it once confirmed no longer needed. Keep the snapshot if you plan to reuse it.

**What this proves:**
- ✅ Manual EBS snapshots capture point-in-time data.
- ✅ Snapshots restore into new, independent volumes.
- ✅ Restored volumes preserve deleted files and older file versions.
- ✅ Live data is unaffected since the restored volume is mounted separately.

---

## Part 5 — Configure Recovery Policies & Disaster Recovery

### Phase 31 – Configure Backup Lifecycle (Retention Policy)

1. **AWS Backup → Backup plans → Daily-EC2-Backup → DailyBackupRule → Edit**.
2. Review/confirm:

| Option | Value |
|---|---|
| Backup Frequency | Daily |
| Start Window | 8 Hours |
| Completion Window | 7 Days |
| Delete After | 30 Days |

Reference retention by environment:

| Environment | Retention |
|---|---|
| Development | 7 Days |
| Testing | 14 Days |
| Production | 30 Days |
| Finance | 365 Days |
| Compliance | 7 Years (per regulation) |

Click **Save**.

### Phase 32 – Configure Cross-Region Backup Copy (DR)

**Step 1 — Create the destination vault:**
1. Switch Region to `ap-southeast-1` (Singapore).
2. **AWS Backup → Backup vaults → Create Backup Vault**.
3. Name: `Singapore-Backup-Vault`. Encryption: AWS managed key. Create.

**Step 2 — Add a copy action in the primary Region:**
1. Switch back to `ap-south-1` (Mumbai).
2. **Backup Plans → Daily-EC2-Backup → Edit** the rule.
3. Scroll to **Copy Actions → Add Copy Action**.

| Setting | Value |
|---|---|
| Destination Region | Singapore |
| Destination Vault | Singapore-Backup-Vault |
| Retention | 30 Days |

4. Click **Save**.

**Step 3 — Verify:**
1. Trigger another on-demand backup (Phase 6).
2. After completion, switch to Singapore → **Backup Vaults → Singapore-Backup-Vault** → confirm a copied recovery point appears.

✅ Cross-Region backup is working.

### Phase 33 – Configure Backup Vault Lock (Ransomware Protection)

1. **AWS Backup → Backup vaults → Production-Backup-Vault → Vault Lock**.
2. Choose a mode:

| Mode | Behavior |
|---|---|
| Governance | Can be removed by authorized IAM users |
| Compliance | Cannot be removed after the grace period — **irreversible** |

> For a lab environment, it's common to review the settings **without** finalizing Compliance mode, since it cannot be undone. Only enable Compliance mode in production once retention requirements are firmly settled.

### Phase 34 – Enable Backup Audit Manager

1. **AWS Backup → Audit Manager → Create Framework**.
2. Name: `AWS Backup Framework`.
3. Typical controls to include:
   - Resources protected
   - Scheduled backups running
   - Backup frequency compliance
   - Retention period compliance
   - Recovery point availability
4. Click **Create Framework**.

### Phase 35 – Recovery Testing (if available in your Region/account)

1. **AWS Backup → Recovery Testing → Create Recovery Testing Plan**.
2. Select the Recovery Point, Target VPC, Target Subnet.
3. AWS periodically performs automated restore tests and generates reports.

### Phase 36 – Define and Document RPO & RTO

**RPO (Recovery Point Objective)** — maximum acceptable data loss.

Example: daily backup at 2 AM; failure occurs at 1 PM → up to 11 hours of data could be lost → **RPO = 24 hours** (bounded by backup frequency).

**RTO (Recovery Time Objective)** — maximum acceptable recovery time.

*Illustrative example only* (general definition — not the same test as Phase 15):

| Event | Time |
|---|---|
| Restore Started | 09:30 |
| Restore Completed | 09:42 |

**Illustrative RTO = 12 minutes**

Document both RPO and your **actual measured RTO from Phase 15** (8 minutes) in your final POC report — don't mix the two.

### Phase 37 – Cost Optimization

- EBS snapshots are **incremental** by default after the first full snapshot.
- Apply lifecycle policies to auto-delete old recovery points.
- Archive long-term backups where the resource type supports cold storage.
- Delete unused manual snapshots regularly.
- Monitor backup storage cost via **AWS Cost Explorer**.

### Phase 38 – Monitoring & Alerting

Monitor with:
- **AWS Backup Dashboard** — job status overview.
- **Amazon CloudWatch** — backup job metrics.
- **Amazon EventBridge** — backup job state-change events.
- **Amazon SNS** — email notifications on failure.

```
AWS Backup
      │
      ▼
EventBridge Rule
      │
      ▼
SNS Topic
      │
      ▼
Email Notification
```

---

## Edge Cases & Gotchas

| Edge Case | Behavior / Fix |
|---|---|
| Restoring EC2 reuses same key pair, but original key file lost | Choose the correct key pair explicitly during restore; you cannot recover a lost private key — plan key management separately |
| Root volume has a partition table | Mount `xvdf1`/`nvme1n1p1`, not the raw device, or mount will fail with "wrong fs type" |
| Nitro-based instance shows NVMe device names | `/dev/sdf` requested at attach time may surface as `/dev/nvme1n1`; always confirm with `lsblk`/`fdisk -l` |
| Compliance-mode Vault Lock | Cannot be reversed or weakened once the grace period ends — test only in Governance mode for labs |
| Cross-Region copy retention differs from source | Configure retention explicitly per copy action; it does not inherit automatically in all cases |
| On-demand backup vs scheduled backup IAM role | Both need a role with `AWSBackupServiceRolePolicyForBackup`; missing permissions cause job failures with an IAM error in job details |
| Restoring into a different AZ | AMI/volume restore can target another AZ, but attaching an EBS snapshot-derived volume requires the **same AZ** as the target instance |
| Deleted resource still shows old recovery points | Recovery points persist per vault retention even after the source EC2 is terminated — clean up manually if not needed |

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Backup job stuck in CREATED | IAM role missing permissions | Verify/attach `AWSBackupDefaultServiceRole` or custom equivalent |
| Restore job fails immediately | Security group/subnet no longer exists | Recreate restore with valid current network settings |
| Mount fails: "special device does not exist" | Wrong device name used | Recheck with `lsblk` — Nitro instances use NVMe naming |
| Mount fails: "wrong fs type, bad option" | Mounted raw disk instead of partition | Mount the partition suffix (e.g., `xvdf1`) |
| Cross-Region recovery point never appears | Copy action not saved, or backup ran before rule was added | Re-trigger an on-demand backup after saving the copy action |
| Vault Lock section greyed out | Insufficient IAM permissions | Ensure `backup:PutBackupVaultLockConfiguration` is granted |
| Snapshot stuck in "pending" for a long time | Large volume / first full snapshot | Wait — first snapshot is a full copy; subsequent ones are incremental and faster |
| Restored instance has no internet access | Public IP not enabled or route table missing | Enable Auto-assign Public IP and confirm subnet has an IGW route |

---



