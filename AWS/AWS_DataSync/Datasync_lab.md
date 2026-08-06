
# AWS DataSync Hands-On Lab
### Migrate On-Premises NFS → S3 using DataSync (Simulated in AWS)

| Detail | Value |
|---|---|
| **Region** | ap-south-1 (Mumbai) |
| **Target** | S3 bucket |
| **Source** | NFS export on EC2 (simulating on-prem) |
| **Agent** | EC2 instance running DataSync Agent AMI |
| **Estimated Time** | 45–60 minutes |
| **Estimated Cost** | ~$0.50–$1.00 (all resources are t3.medium / m5.2xlarge for a short lab) |

---

## Prerequisites Checklist

Before you begin, confirm you have:

- [ ] An AWS account with admin/PowerUser permissions
- [ ] Ability to create EC2, S3, VPC, Security Groups, KMS, CloudWatch, and DataSync resources
- [ ] A key pair created in ap-south-1 (or create one during the lab)
- [ ] AWS CLI installed locally or CloudShell open (for the AMI lookup step)
- [ ] Browser with AWS Console access, logged into ap-south-1

> 💡 **Demo Tip:** Open two browser windows — one for the Console, one for CloudShell/Terminal. This makes it easy to show CLI commands alongside console navigation.

---

## Phase 1 — Networking (≈ 5 min)

### 1.1 Confirm or Create a VPC

1. Go to **VPC Console → Your VPCs**
2. If you already have a default VPC with a public subnet, note its **VPC ID** and **Subnet ID**
3. If not, create one:
   - **VPCs → Create VPC**
   - Name: `datasync-lab-vpc`
   - IPv4 CIDR: `10.0.0.0/16`
   - Select **Create VPC only** (we'll add subnets manually)
   - Then create a **public subnet**: CIDR `10.0.1.0/24`, Availability Zone `ap-south-1a`
   - Attach an **Internet Gateway** to the VPC and route table

### 1.2 Create the Security Group

1. **VPC Console → Security Groups → Create Security Group**
2. Name: `datasync-lab-sg`
3. Description: `Security group for DataSync lab`
4. VPC: select your lab VPC
5. Add inbound rules:

| Type | Protocol | Port | Source | Purpose |
|---|---|---|---|---|
| SSH | TCP | 22 | My IP | SSH access to EC2 instances |
| NFS | TCP | 2049 | `datasync-lab-sg` (self) | NFS data transfer (agent ↔ server) |
| NFS | UDP | 2049 | `datasync-lab-sg` (self) | NFS data transfer (UDP variant) |
| Custom TCP | TCP | 80 | My IP | Temporary — for agent activation key fetch |
| All Traffic | All | All | `datasync-lab-sg` (self) | Simplifies lab communication |

6. Click **Create security group**
7. **Note the Security Group ID** (e.g., `sg-0abc123...`)

> ✅ **Verification:** You should see the security group listed under your VPC's security groups with all 5 inbound rules.

---

## Phase 2 — Launch the NFS Server EC2 (≈ 5 min)

This EC2 instance simulates your on-premises NFS file server.

### 2.1 Launch the Instance

1. **EC2 Console → Instances → Launch Instance**
2. Configure as follows:

| Setting | Value |
|---|---|
| Name | `nfs-server` |
| AMI | **Amazon Linux 2023** (latest) |
| Instance Type | `t3.medium` |
| Key Pair | Select or create `datasync-lab-key` |
| Network Settings | VPC: your lab VPC, Subnet: your public subnet |
| Auto-assign Public IP | **Enable** |
| Security Group | Select `datasync-lab-sg` |
| Storage | 20 GB gp3 (default is fine) |

3. Click **Launch Instance**
4. Wait until status shows **Running** and **2/2 checks passed** (~2 min)

### 2.2 Configure the NFS Export

1. Select the `nfs-server` instance → click **Connect** → **EC2 Instance Connect** → **Connect**
2. Run the following commands:

```bash
# Install NFS server packages
sudo dnf install -y nfs-utils

# Create the export directory
sudo mkdir -p /export/engineering-docs
sudo chmod 777 /export/engineering-docs

# Create 20 sample test files
sudo bash -c 'for i in $(seq 1 20); do echo "Sample CAD file $i" > /export/engineering-docs/drawing-$i.txt; done'

# Verify files were created
ls -la /export/engineering-docs/ | wc -l
# Expected output: 23  (20 files + . + .. + total line)
```

3. Configure the NFS export:

```bash
# Add export entry
sudo bash -c 'echo "/export/engineering-docs *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports'

# Apply exports
sudo exportfs -ra

# Enable and start NFS server
sudo systemctl enable --now nfs-server

# Verify NFS is running
sudo systemctl status nfs-server --no-pager
# Expected: active (running)

# Verify export is visible
sudo exportfs -v
# Expected: /export/engineering-docs  <world>
```

4. **Note the Private IP** of `nfs-server`:

```bash
hostname -I
# Example output: 10.0.1.50
```

> ✅ **Verification:** You should see the NFS server running and the export path visible via `exportfs -v`. The private IP is your **Source NFS server address** for DataSync.

> 💡 **Demo Talking Point:** "This EC2 instance is standing in for our real on-prem NFS server. In production, the DataSync Agent would be deployed physically close to this server to minimize latency."

---

## Phase 3 — Get the DataSync Agent AMI (≈ 2 min)

### 3.1 Look Up the AMI ID

1. Open **AWS CloudShell** (click the terminal icon in the top-right of the Console)
2. CloudShell auto-opens in your current region. Confirm it's `ap-south-1`:

```bash
aws configure get region
# Expected: ap-south-1
```

3. Retrieve the DataSync Agent AMI ID:

**For Basic mode (recommended for this lab):**
```bash
aws ssm get-parameter \
  --name /aws/service/datasync/ami \
  --region ap-south-1 \
  --query Parameter.Value \
  --output text
```

**For Enhanced mode (optional, if you want to try it):**
```bash
aws ssm get-parameter \
  --name /aws/service/datasync/ami/v3 \
  --region ap-south-1 \
  --query Parameter.Value \
  --output text
```

4. **Copy the AMI ID** (e.g., `ami-0a1b2c3d4e5f6789a`) — you'll need it in the next step.

> ✅ **Verification:** You should get a valid AMI ID string back. If you get an error, check that CloudShell is in `ap-south-1` and that your IAM user has `ssm:GetParameter` permissions.

---

## Phase 4 — Launch the DataSync Agent EC2 (≈ 5 min)

### 4.1 Launch the Agent Instance

1. **EC2 Console → Launch Instance**
2. Use the URL builder for the DataSync AMI:

```
https://console.aws.amazon.com/ec2/v2/home?region=ap-south-1#LaunchInstanceWizard:ami=<YOUR_AMI_ID>
```

Replace `<YOUR_AMI_ID>` with the AMI from Step 3. Paste this URL in your browser.

3. Configure the instance:

| Setting | Value |
|---|---|
| Name | `datasync-agent` |
| AMI | Pre-filled with DataSync Agent AMI |
| Instance Type | `m5.2xlarge` (Basic mode) |
| Key Pair | Same as `nfs-server` |
| Network Settings → VPC/Subnet | Same as `nfs-server` |
| Auto-assign Public IP | **Enable** |
| Security Group | `datasync-lab-sg` |
| Storage | 50 GB gp3 (DataSync needs space for its working directory) |

4. Click **Launch Instance**
5. Wait for status **Running** (~2 min)

### 4.2 Verify Port 80 is Reachable

The DataSync Agent listens on port 80 for activation. From your local machine:

```bash
# Replace with the public IP of datasync-agent
curl -I http://<PUBLIC_IP_OF_AGENT>:80
# Expected: HTTP/1.1 200 OK (or 403/401 — any response means port is open)
```

> ✅ **Verification:** You get an HTTP response from the agent's public IP on port 80. If not, double-check the security group has TCP 80 open from your IP.

> 💡 **Demo Talking Point:** "The DataSync Agent is a lightweight VM image that AWS publishes. In production, you'd deploy it as a VMware/KVM VM on your physical hardware. Here we run it on EC2 for the lab."

---

## Phase 5 — Activate the DataSync Agent (≈ 3 min)

### 5.1 Create the Agent in the Console

1. Go to **DataSync Console → Agents → Create agent**
2. Configure:

| Setting | Value |
|---|---|
| Service endpoint | **Public service endpoints in ap-south-1** |
| Activation key | **Automatically get the activation key from your agent** |
| Agent address | Public IP of `datasync-agent` instance |
| Agent name | `nfs-migration-agent` |

3. Click **Create agent**
4. The agent may briefly show **OFFLINE** — this is normal. Wait 1–2 minutes for it to transition to **ONLINE**

> ✅ **Verification:** Agent status shows **ONLINE** with a green indicator.

> ⚠️ **Troubleshooting:**
> - If activation fails: confirm port 80 is open in the security group, confirm the agent instance is running, and confirm you're using the public IP (not private)
> - Alternative: if "Get key" doesn't work, SSH into the agent instance and run `sudo cat /etc/datasync/activation-key` to get the key manually, then select **Manually enter activation key**

---

## Phase 6 — Create Source Location (NFS) (≈ 3 min)

1. **DataSync Console → Locations → Create location**
2. Configure:

| Setting | Value |
|---|---|
| Location type | **NFS** |
| Agent | `nfs-migration-agent` |
| NFS server | **Private IP** of `nfs-server` (from Phase 2) |
| Mount path | `/export/engineering-docs` |
| Mount options | Leave defaults |

3. Click **Create location**
4. Wait for creation to complete

> ✅ **Verification:** Location appears in the list with status **Available**.

---

## Phase 7 — Create Destination Location (S3) (≈ 5 min)

### 7.1 Create the S3 Bucket

1. **S3 Console → Create bucket**
2. Configure:

| Setting | Value |
|---|---|
| Bucket name | `brightwave-engineering-docs-<YOUR_ACCOUNT_ID>` (replace with your 12-digit account ID) |
| Region | ap-south-1 (Mumbai) |
| Object Ownership | ACLs disabled (recommended) |
| Block Public Access | Keep all checked (block public) |
| Versioning | **Enable** |
| Encryption | **SSE-KMS**, create a new key or use the default AWS-managed key |
| Bucket policy | Leave empty |

3. Click **Create bucket**

### 7.2 Create the S3 DataSync Location

1. **DataSync Console → Locations → Create location**
2. Configure:

| Setting | Value |
|---|---|
| Location type | **S3** |
| S3 bucket | Select `brightwave-engineering-docs-<ACCOUNT_ID>` |
| Folder | `/nfs-migration/` |
| Storage class | S3 Standard (default) |
| Access tier | None |
| IAM role | **Create new role** (DataSync auto-creates a least-privilege role) |

3. Click **Create location**

> ✅ **Verification:** Both locations (NFS and S3) appear in the Locations list with status **Available**.

---

## Phase 8 — Create the DataSync Task (≈ 5 min)

### 8.1 Configure the Task

1. **DataSync Console → Tasks → Create task**
2. Configure:

| Setting | Value |
|---|---|
| Task name | `nfs-to-s3-nightly-sync` |
| Source location | NFS location from Phase 6 |
| Destination location | S3 location from Phase 7 |
| Task mode | **Basic** (matches the agent mode from Phase 4) |
| Transfer mode | **Transfer only data that has changed** |
| Verification mode | **Verify only transferred data (recommended)** |
| Overwrite files | **Always** |
| Preserve metadata | ✅ Ownership, ✅ POSIX permissions, ✅ Timestamps |
| Filters | None (transfer everything) |
| Schedule | **Unscheduled** (we'll add one in Phase 12) |
| Task logging | **CloudWatch Logs** → Create log group `/aws/datasync` |

3. Click **Create task**
4. Wait for task status to show **Available** (refresh if needed)

> ✅ **Verification:** Task appears in the Tasks list with status **Available**.

> 💡 **Demo Talking Point:** "Notice 'Transfer only data that has changed' — this is the key to incremental sync. DataSync uses checksums to detect what's actually changed, not just file timestamps."

---

## Phase 9 — Run the Initial Transfer (≈ 5 min)

### 9.1 Start the Task

1. Click on the task `nfs-to-s3-nightly-sync`
2. Click **Start** → **Start with defaults**
3. The task enters **Running** state
4. Click **See execution details** to watch real-time progress

### 9.2 Monitor the Transfer

Watch for these in the execution details:
- **Files transferred:** should show 20
- **Bytes transferred:** ~400 bytes (20 small text files)
- **Duration:** should complete in under a minute
- **Status:** should show **SUCCESS**

### 9.3 Verify in S3

1. Go to **S3 Console** → open `brightwave-engineering-docs-<ACCOUNT_ID>`
2. Navigate to the `/nfs-migration/` prefix
3. Confirm all 20 files (`drawing-1.txt` through `drawing-20.txt`) are present

```bash
# Optional: verify via CLI
aws s3 ls s3://brightwave-engineering-docs-<ACCOUNT_ID>/nfs-migration/ --region ap-south-1
# Expected: 20 objects listed
```

> ✅ **Verification:** All 20 files landed in S3 under `/nfs-migration/`. Execution shows SUCCESS with 20 files transferred.

---

## Phase 10 — Test Incremental Sync (≈ 3 min)

This is the most important verification step — it proves DataSync only transfers changes.

### 10.1 Modify the NFS Source

1. SSH back into `nfs-server` (or use EC2 Instance Connect)
2. Make changes:

```bash
# Modify an existing file
sudo bash -c 'echo "New revision - updated content" > /export/engineering-docs/drawing-1.txt'

# Add a brand new file
sudo bash -c 'echo "Brand new file added after initial sync" > /export/engineering-docs/drawing-21.txt'

# Verify changes
ls -la /export/engineering-docs/ | wc -l
# Expected: 23  (21 files + . + .. + total line)
```

### 10.2 Run the Task Again

1. Go to **DataSync Console → Tasks** → select `nfs-to-s3-nightly-sync` → **Start** → **Start with defaults**
2. Watch execution details

### 10.3 Verify Incremental Behavior

- **Expected:** Only **2 files** transferred (drawing-1.txt was modified, drawing-21.txt is new)
- **NOT expected:** All 21 files transferred again

```bash
# Verify in S3
aws s3 ls s3://brightwave-engineering-docs-<ACCOUNT_ID>/nfs-migration/ --region ap-south-1
# Expected: 21 objects (20 original + 1 new)

# Verify the modified file content
aws s3 cp s3://brightwave-engineering-docs-<ACCOUNT_ID>/nfs-migration/drawing-1.txt - --region ap-south-1
# Expected output: "New revision - updated content"
```

> ✅ **Verification:** Execution shows 2 files transferred, not 21. This confirms incremental sync is working correctly.

> 💡 **Demo Talking Point:** "This is the power of DataSync — it uses content-level checksums to detect actual changes. Even though we ran the full task again, only the 2 changed files moved. For a dataset with millions of files where only a handful change nightly, this saves enormous time and cost."

---

## Phase 11 — Schedule Nightly Syncs (≈ 3 min)

1. **DataSync Console → Tasks** → select `nfs-to-s3-nightly-sync` → **Edit**
2. Under **Schedule**, click **Edit**
3. Choose **Recurring schedule**
4. Enter cron expression:

```
0 2 * * ? *
```

This runs the task at **2:00 AM UTC daily** (which is 7:30 AM IST in ap-south-1).

5. Click **Save**
6. The task now shows a schedule icon and will execute automatically

> ✅ **Verification:** The task's schedule column shows the cron expression. The next scheduled run time is visible.

> 💡 **Demo Talking Point:** "In production, you'd typically run this during off-peak hours. The cron expression `0 2 * * ? *` means every day at 2 AM UTC. DataSync handles the rest — it creates a new execution each time."

---

## Phase 12 — Monitor & Audit (≈ 3 min)

### 12.1 CloudWatch Logs

1. **CloudWatch Console → Log groups** → find `/aws/datasync`
2. Open the log stream for the latest task execution
3. Confirm you see per-file transfer entries

### 12.2 CloudWatch Metrics

1. **CloudWatch Console → Metrics** → select **DataSync** namespace
2. Look for these metrics:
   - `BytesTransferred`
   - `FilesTransferred`
   - `FilesVerified`
   - `TaskExecutionDuration`

### 12.3 CloudTrail (Audit Trail)

1. **CloudTrail Console → Event history**
2. Event source: filter by `datasync.amazonaws.com`
3. You should see events like:
   - `CreateTask`
   - `StartTaskExecution`
   - `CreateLocationNfs`
   - `CreateLocationS3`

> ✅ **Verification:** You can see logs, metrics, and audit events all populated from the DataSync operations.

---

## Phase 13 — Cleanup (≈ 5 min)

> ⚠️ **Important:** Delete in this exact order to avoid dependency errors.

### Step-by-step cleanup:

| Order | Action | Console Path |
|---|---|---|
| 1 | Delete the DataSync task | DataSync → Tasks → Select task → Delete |
| 2 | Delete the NFS location | DataSync → Locations → Select NFS → Delete |
| 3 | Delete the S3 location | DataSync → Locations → Select S3 → Delete |
| 4 | Delete the DataSync agent | DataSync → Agents → Select agent → Delete agent |
| 5 | Terminate the agent EC2 | EC2 → Instances → Select `datasync-agent` → Instance state → Terminate |
| 6 | Terminate the NFS server EC2 | EC2 → Instances → Select `nfs-server` → Instance state → Terminate |
| 7 | Empty and delete the S3 bucket | S3 → Select bucket → Empty → Delete bucket |
| 8 | Delete the KMS key (if custom) | KMS → Customer managed keys → Select key → Schedule deletion |
| 9 | Delete the CloudWatch log group | CloudWatch → Log groups → `/aws/datasync` → Delete |
| 10 | Delete the security group | VPC → Security Groups → `datasync-lab-sg` → Delete |
| 11 | Delete the VPC (if created for lab) | VPC → Your VPCs → Select VPC → Delete VPC |

> 💡 **Demo Talking Point:** "Always clean up lab resources. The most common mistake is forgetting to empty versioned S3 buckets before deletion — you must delete all object versions first."

---

## Quick Reference: Expected Results Summary

| Step | Expected Result |
|---|---|
| NFS server running | `systemctl status nfs-server` → active (running) |
| 20 test files created | `ls /export/engineering-docs/` shows 20 files |
| Agent ONLINE | DataSync Console → Agents → green indicator |
| Source location created | Available in Locations list |
| Destination location created | Available in Locations list |
| Initial transfer | 20 files, ~400 bytes, status SUCCESS |
| S3 verification | 20 objects under `/nfs-migration/` |
| Incremental sync | Only 2 files transferred (not 21) |
| Scheduled task | Cron `0 2 * * ? *` visible on task |
| CloudWatch logs | Per-file entries in `/aws/datasync` |
| CloudTrail events | `CreateTask`, `StartTaskExecution` visible |

---

## Troubleshooting Quick Reference

| Problem | Likely Cause | Fix |
|---|---|---|
| Agent stays OFFLINE after creation | Port 80 blocked by SG or agent not running | Check SG has TCP 80 from your IP; verify agent instance is running |
| NFS location creation fails | Agent can't reach NFS server | Confirm both instances are in the same VPC/subnet; verify SG allows NFS (2049) self-referencing |
| Task shows "Error" on start | IAM role missing or S3 bucket doesn't exist | Let DataSync auto-create the role; verify bucket exists |
| Incremental sync transfers all files | Source files were modified during transfer or checksum mismatch | Re-run task; check if files are being actively written |
| S3 bucket deletion fails | Versioning enabled, objects still exist | Empty bucket including all versions first |
| Activation key fetch fails | Browser can't reach agent on port 80 | Test with `curl http://<agent-public-ip>:80` from your local machine |

---

## Demo Script (If Presenting)

| Time | What to Show | Talking Point |
|---|---|---|
| 0–5 min | VPC + SG setup | "We're creating isolated networking for the lab" |
| 5–10 min | NFS server EC2 launch | "This simulates our on-prem file server" |
| 10–12 min | NFS export + test files | "20 sample engineering documents ready to migrate" |
| 12–15 min | Agent AMI lookup + EC2 launch | "The DataSync Agent runs as a lightweight VM" |
| 15–18 min | Agent activation | "The agent registers with the DataSync service" |
| 18–22 min | Source + destination locations | "Pointing DataSync at the NFS source and S3 destination" |
| 22–25 min | Task creation | "Configured for incremental, verified transfers" |
| 25–30 min | Initial transfer run | "Watch all 20 files move in real time" |
| 30–33 min | Incremental sync test | "Only 2 files transfer — the power of change detection" |
| 33–35 min | Schedule setup | "Nightly automated syncs, no manual intervention" |
| 35–40 min | Monitoring (CloudWatch + CloudTrail) | "Full visibility and audit trail" |
| 40–45 min | Cleanup | "Always clean up lab resources" |

---

## Notes for Instructors / Team Leads

- **Pre-requisites to verify before lab:** Confirm all participants have AWS account access with sufficient permissions, and that the `ap-south-1` region is enabled in their account.
- **Time management:** If short on time, skip Enhanced mode and VPC endpoint setup. Focus on Phases 1–10 for the core experience.
- **Cost optimization:** For teams, consider using Spot Instances for the NFS server and Agent EC2 (they're short-lived). The m5.2xlarge can also be down-sized if cost is a concern (but the minimum is 2xlarge).
- **Extending the lab:** Advanced learners can try:
  - Adding **S3 Transfer Acceleration** to the destination
  - Configuring **VPC endpoints** for the DataSync service (instead of public endpoints)
  - Testing with an **SMB source** instead of NFS
  - Adding **filter rules** (e.g., only transfer `*.txt` files, or exclude specific prefixes)
  - Testing **bandwidth limits** on the task to simulate constrained network links

---

## Production Architecture Notes (For Discussion)

| Lab Scenario | Production Reality |
|---|---|
| EC2 instance hosts NFS | Physical NFS server in your datacenter |
| EC2 instance hosts DataSync Agent | VMware/KVM/Hyper-V VM or physical server onsite |
| Public internet for DataSync service endpoints | VPC endpoints (PrivateLink) for security and performance |
| t3.medium NFS server | Enterprise NAS/SAN with 10Gb+ networking |
| Basic mode agent | Enhanced mode agent for multi-threaded transfers |
| Manual cleanup | Infrastructure-as-Code (Terraform/CloudFormation) with automated teardown |
| Single task | Multiple tasks with different filters, schedules, and destinations |

---

*Lab created for hands-on practice with AWS DataSync. Always clean up resources after completion to avoid unexpected AWS charges.*

**Disclaimer:** This lab uses EC2 instances to simulate on-premises infrastructure. AWS recommends deploying the DataSync Agent as close to the source data as possible in production environments (i.e., on-premises, not in EC2) to minimize network latency.
