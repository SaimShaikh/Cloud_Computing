
# AWS DataSync — End-to-End Lab Guide

---

## 1. Scenario

**Company:** BrightWave Engineering Pvt Ltd, Pune
**Problem:** The engineering team stores 50 TB of CAD drawings, build logs, and documentation on an **on-premises Linux NFS file server**. Manual `rsync` jobs to AWS S3 have been failing silently, there's no retry logic, no integrity verification, and no visibility into what actually transferred each night.

**Goal:** Replace the fragile `rsync` script with **AWS DataSync** to:

- Migrate the initial 50 TB dataset to Amazon S3 (Standard-IA, versioned, SSE-KMS encrypted)
- Run nightly **incremental** syncs so only changed files are transferred
- Preserve file ownership, permissions, and timestamps
- Get CloudWatch metrics + CloudTrail audit logs for every transfer
- Eventually feed the S3 data lake into Athena for reporting

**Lab constraint:** We don't have real on-prem hardware, so we **simulate the on-prem NFS server using an EC2 instance** in a "corporate" VPC, and deploy the **DataSync Agent** as a second EC2 instance — a common way to practice DataSync end-to-end entirely inside AWS.

> **Important:** AWS explicitly does **not recommend** deploying the Agent on EC2 when the real source is genuinely on-premises, because of the added network latency of routing on-prem traffic through the cloud first. In production, you'd deploy the Agent as a VMware/KVM/Hyper-V VM physically inside your datacenter, as close to the NFS/SMB server as possible. The EC2-based NFS server + EC2-based Agent setup below is purely a **lab simulation** so you can practice the workflow without physical hardware.

**What you will build:**

| Component | Real-world equivalent | Lab substitute |
|---|---|---|
| On-prem NFS server | Corporate file server | EC2 instance running `nfs-kernel-server` |
| DataSync Agent | VM on VMware/Hyper-V/KVM | DataSync Agent AMI on EC2 |
| Destination | AWS S3 | Amazon S3 bucket (versioned, SSE-KMS) |

---

## 2. Architecture Diagram

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/0345e457-c7b5-4b0f-943a-1ce2e312b3cd" />


**Data flow during a task execution:**

```
Read Files → Split into Chunks → Compress → Encrypt (TLS)
    → Parallel Transfer → AWS DataSync Service
    → Write to S3 → Checksum Validation → Task Execution: SUCCESS
```

---

## 3. Concept Deep Dive

### 3.1 What is AWS DataSync?

AWS DataSync is a **managed data transfer service** that automates moving large amounts of data between on-premises storage, AWS storage services, other cloud providers, AWS Regions, and AWS accounts — replacing brittle manual tools like `rsync`, `scp`, and `sftp` scripts with a service that handles scheduling, retries, encryption, and integrity verification automatically.

### 3.2 Why DataSync Instead of Manual Scripts?

| Problem with manual `rsync`/scripts | How DataSync solves it |
|---|---|
| Slow, single-threaded | Parallel, multi-threaded transfer |
| Script failures go unnoticed | Automatic retries + CloudWatch alarms |
| No scheduling built in | Native hourly/daily/weekly/monthly scheduling |
| No integrity verification | Checksum + metadata comparison after every transfer |
| No audit trail | CloudTrail logs every API call |
| Re-copies everything every time | Incremental sync — only changed files move |

### 3.3 Core Components

| Component | What it is |
|---|---|
| **Agent** | A VM (or EC2 instance) that acts as the bridge between storage AWS cannot directly reach and the DataSync service |
| **Location** | A source or destination endpoint (NFS share, SMB share, S3 bucket, EFS, FSx, object storage) |
| **Task** | The definition of a transfer job: source, destination, schedule, filters, verification settings, overwrite/delete behavior |
| **Task Execution** | One actual run of a task — has its own logs, throughput, errors, and duration |

**Agent — the bridge**
The Agent is a virtual appliance that reads from (or writes to) your storage system, compresses and encrypts the data, and hands it off to the DataSync service. It's not long-term storage — it only buffers data in memory/disk while a transfer is in progress. Think of it as a smart transfer gateway rather than a proxy server.

**Location — where data comes from or goes to**
A Location is simply an endpoint definition. Examples:
- *Source locations:* an NFS export, an SMB share, an existing S3 bucket, an EFS file system
- *Destination locations:* an S3 bucket, an EFS file system, an FSx file system, another NFS/SMB share

**Task — the job definition**
A Task ties a source Location to a destination Location and defines *how* the transfer should behave. Example: "Copy `/export/engineering-docs` (NFS) to `s3://brightwave-engineering-docs/nfs-migration/` every night at 2 AM, preserving permissions, verifying only transferred data."

**Task Execution — one run of the job**
Every time a Task runs — whether manually or on schedule — DataSync creates a new Task Execution record. Each execution has its own start/end time, throughput graph, per-file transfer log, and success/error status, which is what you'll inspect in CloudWatch after every run.

### 3.4 Is an Agent Always Required?

| Transfer | Agent required? |
|---|---|
| On-prem NFS/SMB → S3 | ✅ Yes |
| On-prem → EFS / FSx | ✅ Yes |
| S3 → S3 | ❌ No |
| EFS → EFS (same account/region) | ❌ No |
| EFS → S3 | ❌ No |
| S3 → FSx | ❌ No |
| Azure Blob → S3 | Sometimes (depends on task mode) |
| Google Cloud Storage → S3 | Yes (Basic mode) |

### 3.5 Agent Modes

| Mode | Notes |
|---|---|
| **Basic Mode** | Supports NFS, SMB, HDFS, object storage, Azure Blob. Has file-count guidelines (e.g. ~20 million files per task as a general planning limit) |
| **Enhanced Mode** | Newer mode, optimized for S3 workflows, faster preparation, no file-count quota. Requires an Enhanced-mode agent image |

**Agent VM resource requirements (per AWS documentation):**

| Resource | Basic mode | Enhanced mode |
|---|---|---|
| Virtual processors | 4 | 8 |
| Disk space | 80 GB | 80 GB |
| RAM | 32 GB (up to 20M files) / 64 GB (more than 20M files) | 32 GB |
| Recommended EC2 instance | `m5.2xlarge` (up to 20M files) / `m5.4xlarge` (more) | `m6a.2xlarge` |

The minimum EC2 instance size for any DataSync agent is **2xlarge**.

### 3.6 Agent Workflow — What Happens During a Transfer

This is the internal sequence the Agent follows on every task execution:

```
Step 1
Agent scans the source.

NFS
SMB
Object Storage

↓

Step 2
Collects metadata.

Example:
  Filename
  Size
  Timestamp
  Owner
  Permissions

↓

Step 3
Compares source and destination.

↓

Step 4
Transfers only the required files (depending on task settings).

↓

Step 5
Compresses the data.

↓

Step 6
Encrypts the transfer using TLS.

↓

Step 7
Uploads to the DataSync service.

↓

Step 8
DataSync writes the data to S3, EFS, or FSx.

↓

Step 9
Validates checksums to confirm integrity.
```

**Agent communication model:**
```
Agent  ──outbound TCP 443 (HTTPS)──▶  AWS DataSync Endpoint
```
No inbound internet access is required for normal (post-activation) operation — the Agent always initiates the connection outward.

**Buffering, not storage:** the Agent temporarily buffers data in memory/disk mid-transfer, but it does not retain your files permanently — once a transfer completes, nothing is kept on the Agent itself.

### 3.7 Agent Activation Workflow

```
Deploy Agent (VM or EC2)
        │
Boot the Agent
        │
Agent obtains an IP address
        │
Open DataSync Console → Create agent
        │
Enter the agent's IP address (or fetch key via local console/curl)
        │
AWS validates connectivity on port 80 and issues an Activation Key
        │
Console/CLI submits the Activation Key → CreateAgent API call
        │
Agent is registered to your AWS account + Region
        │
Port 80 is closed automatically; ongoing comms use outbound 443
        │
        ▼
   Agent status: ONLINE
```

Once activated, AWS fully manages the Agent's software updates and patching — you don't maintain it like a normal VM.

### 3.8 End-to-End DataSync Task Workflow

The full journey from zero to a running, scheduled migration:

```
Step 1   Deploy the Agent (VM or EC2)
             │
Step 2   Activate the Agent
             │
Step 3   Create the Source Location (e.g., NFS)
             │
Step 4   Create the Destination Location (e.g., S3)
             │
Step 5   Create the Task (source + destination + settings)
             │
Step 6   Run the Task (on-demand, for the first full copy)
             │
Step 7   Verify results (CloudWatch metrics, S3 contents)
             │
Step 8   Attach a recurring Schedule for ongoing incremental syncs
             │
Step 9   Monitor every execution going forward (CloudWatch + CloudTrail)
```

This is exactly the sequence the hands-on lab in Section 4 walks through.

### 3.9 Transfer Modes

```
Initial Transfer:  100 TB  →  Copy everything
Incremental Sync:  Only files changed since last run  →  e.g. 100 GB changed → only 100 GB transferred
```

### 3.10 Data Integrity Verification

Every task execution compares **file size, metadata, and checksum** between source and destination to guarantee the copy is byte-identical — not just "file exists."

### 3.11 Encryption

| Layer | Mechanism |
|---|---|
| In transit | TLS over HTTPS (agent ↔ DataSync service) |
| At rest (S3) | SSE-S3 or SSE-KMS |
| At rest (EFS) | Encryption at rest |
| At rest (FSx) | KMS encryption |

### 3.12 Filtering, Scheduling, Bandwidth Throttling

```
Include filter:   *.pdf, /project1/*
Exclude filter:   temp/, logs/, cache/
Schedule:         Daily @ 2:00 AM
Bandwidth cap:    Limit DataSync to 200 Mbps out of a 1 Gbps office link
```

### 3.13 Metadata Preservation

DataSync can preserve ownership, POSIX permissions, timestamps, and ACLs (where the destination supports them) — critical for engineering file shares where permission structure matters.

### 3.14 DataSync vs Alternatives

| Feature | AWS DataSync | rsync |
|---|---|---|
| Managed | ✅ | ❌ |
| Scheduling | ✅ | Manual (cron) |
| Verification | ✅ Checksum | Basic |
| Monitoring | ✅ CloudWatch | ❌ |
| Retries | Automatic | Manual |

| | DataSync | Snowball |
|---|---|---|
| Transport | Network | Physical device shipped |
| Best for | Online, ongoing migration | Offline, one-time bulk transfer when network is too slow |
| Sync | Continuous/incremental | One-time import/export |

| | DataSync | Storage Gateway |
|---|---|---|
| Purpose | Moves/migrates data | Provides ongoing hybrid storage access |
| Nature | Task-based, finite job | Continuous caching + backup layer |

---

## 4. Hands-On Lab (Console-First)

> All steps use the **AWS Management Console**. Region used in this lab: **ap-south-1 (Mumbai)**.

### Step 1 — Create the VPC networking (if not already present)

1. Go to **VPC Console → Your VPCs** and confirm you have a VPC with at least one **public subnet** (both EC2 instances will sit here for lab simplicity).
2. Note the VPC ID and Subnet ID — you'll need them for both EC2 instances.
3. Go to **Security Groups → Create security group**, name it `datasync-lab-sg`, and add these inbound rules:
   - SSH (22) from your IP
   - NFS (2049, TCP+UDP) from the security group itself (self-referencing, so agent ↔ NFS server can talk)
   - All traffic from the security group itself (self-referencing) — simplifies the lab

### Step 2 — Launch the simulated on-prem NFS server (EC2)

1. **EC2 Console → Launch Instance**
2. Name: `nfs-server`
3. AMI: **Amazon Linux 2023**
4. Instance type: `t3.medium`
5. Key pair: select or create one
6. Network: your lab VPC, public subnet, **Auto-assign public IP: Enable**
7. Security group: `datasync-lab-sg`
8. Storage: 20 GB gp3 (this simulates the NFS-exported volume)
9. Launch the instance and wait for it to reach **Running**

### Step 3 — Configure the NFS export

1. Connect to `nfs-server` via **EC2 Instance Connect** (Console → Connect) or SSH.
2. Install and start the NFS server:
   ```
   sudo dnf install -y nfs-utils
   sudo mkdir -p /export/engineering-docs
   sudo chmod 777 /export/engineering-docs
   ```
3. Create some sample test files:
   ```
   sudo bash -c 'for i in $(seq 1 20); do echo "Sample CAD file $i" > /export/engineering-docs/drawing-$i.txt; done'
   ```
4. Configure the export in `/etc/exports`:
   ```
   sudo bash -c 'echo "/export/engineering-docs *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports'
   sudo exportfs -ra
   sudo systemctl enable --now nfs-server
   ```
5. Note the **private IP** of `nfs-server` (used as the DataSync source location).

### Step 4 — Get the DataSync Agent AMI ID

Unlike VMware/KVM/Hyper-V (where you download an image file from the console), the **EC2 agent is launched from an AWS-published AMI**, which you look up via AWS CLI or CloudShell:

1. Open **AWS CloudShell** (or a terminal with AWS CLI configured) in the same region as your lab (ap-south-1).
2. Run, for a **Basic mode** agent:
   ```
   aws ssm get-parameter --name /aws/service/datasync/ami --region ap-south-1
   ```
   or for an **Enhanced mode** agent:
   ```
   aws ssm get-parameter --name /aws/service/datasync/ami/v3 --region ap-south-1
   ```
3. Note the `Value` field — this is the AMI ID, e.g. `ami-0123456789abcdef0`.

### Step 5 — Launch the Agent EC2 Instance

1. Build this URL, substituting your region and the AMI ID from Step 4:
   ```
   https://console.aws.amazon.com/ec2/v2/home?region=ap-south-1#LaunchInstanceWizard:ami=ami-0123456789abcdef0
   ```
2. Paste it into your browser — this opens the EC2 launch wizard pre-filled with the DataSync AMI.
3. Name: `datasync-agent`
4. Instance type: `m5.2xlarge` (Basic mode) or `m6a.2xlarge` (Enhanced mode) — 2xlarge is the minimum supported size
5. Key pair: select one
6. Network settings → Edit:
   - VPC/subnet: same as `nfs-server`
   - Auto-assign public IP: **Enable** (needed for this lab's activation approach)
   - Security group: `datasync-lab-sg`, plus an inbound rule allowing **TCP 80 from your own IP** (required temporarily so your browser can fetch the activation key — DataSync closes this port automatically once activation completes)
7. Launch the instance, wait for **Running**, and note its **public IP address**.

### Step 6 — Activate the Agent

1. **DataSync Console → Agents → Create agent**
2. Under **Service endpoint**, choose **Public service endpoints in your current AWS Region** (simplest option for this lab; VPC endpoints are used for private-only setups)
3. Under **Activation key**, choose **Automatically get the activation key from your agent**
4. For **Agent address**, enter the `datasync-agent` instance's public IP, then click **Get key**
   - This works because your browser reaches the agent on port 80, which you opened in Step 5.
5. Agent name: `nfs-migration-agent`
6. Click **Create agent**

> Immediately after creation the agent may briefly show **OFFLINE** — this is expected and it should move to **ONLINE** within a minute or two.

> **If activation fails:** confirm your browser can reach the agent's public IP on port 80 (check the security group rule), or use the alternative "manually enter activation key" option by retrieving the key from the agent's local console instead.

### Step 6 — Create the Source Location (NFS)

1. **DataSync Console → Locations → Create location**
2. Location type: **NFS**
3. Agent: select `nfs-migration-agent`
4. NFS server: the **private IP** of `nfs-server`
5. Mount path: `/export/engineering-docs`
6. Click **Create location**

### Step 7 — Create the Destination Location (S3)

1. First, create the destination bucket: **S3 Console → Create bucket**
   - Name: `brightwave-engineering-docs-<your-account-id>`
   - Enable **Versioning**
   - Encryption: **SSE-KMS**, choose or create a customer-managed key
2. Back in **DataSync Console → Locations → Create location**
3. Location type: **S3**
4. S3 bucket: select the bucket you just created
5. Folder: `/nfs-migration/`
6. IAM role: let DataSync **auto-create** the role (grants least-privilege S3 access)
7. Click **Create location**

### Step 8 — Create the Task

1. **DataSync Console → Tasks → Create task**
2. Source location: the NFS location from Step 6
3. Destination location: the S3 location from Step 7
4. **Settings:**
   - Verify data: **Verify only data transferred**
   - Metadata: preserve ownership, permissions, timestamps
   - Filters: none (transfer everything for this lab)
   - Schedule: **Run on demand** for the first test run
   - Task logging: send logs to CloudWatch Logs (create/select a log group)
5. Review and **Create task**

### Step 9 — Run the Initial Transfer

1. Open the task, click **Start → Start with defaults**
2. Watch the **Task Execution** detail page — it shows files transferred, bytes transferred, duration, and throughput in real time.
3. Once status shows **SUCCESS**, verify in the S3 console that all 20 sample files landed under `/nfs-migration/`.

### Step 10 — Test Incremental Sync

1. Back on `nfs-server`, add a new file and modify an existing one:
   ```
   sudo bash -c 'echo "New revision" > /export/engineering-docs/drawing-1.txt'
   sudo bash -c 'echo "Brand new file" > /export/engineering-docs/drawing-21.txt'
   ```
2. Run the task again from the console.
3. Confirm the execution log shows only **2 files transferred**, not all 21 — this proves incremental sync is working.

### Step 11 — Schedule Nightly Syncs

1. Open the task → **Edit**
2. Under **Schedule**, choose a recurring schedule (e.g., cron `0 2 * * ? *` for 2:00 AM daily)
3. Save — the task will now run automatically every night.

### Step 12 — Verify Monitoring & Audit

1. **CloudWatch → Log groups** → open the DataSync log group → confirm per-file transfer logs appear.
2. **CloudWatch → Metrics → DataSync** → check `BytesTransferred`, `FilesTransferred`, `FilesVerified`.
3. **CloudTrail → Event history** → filter by event source `datasync.amazonaws.com` → confirm `CreateTask`, `StartTaskExecution` events are logged.

---

## 5. Edge Cases

| Edge case | What happens | How to handle it |
|---|---|---|
| Agent goes offline mid-task | Task execution fails/stalls | Task retries automatically; investigate agent's network/security group after |
| Source file deleted after task starts but before it's read | May show as a transient error in that execution | Re-run task; DataSync will reconcile on next incremental run |
| Destination file modified out-of-band (someone edits directly in S3) | Next incremental sync may overwrite it depending on task's overwrite settings | Set the task's "Overwrite files" behavior explicitly; avoid manual edits at the destination |
| Multiple agents assigned to one location | You can associate up to **8 agents** with a single NFS/SMB/HDFS location to scale throughput | Add agents via **Locations → Edit → Agents**; verify all assigned agents are online before large transfers, since an offline agent in the pool can affect that location's tasks |
| Very large file counts (millions of small files) | Slower listing/scanning phase | Consider splitting into multiple tasks by subdirectory, or use multiple agents in parallel |
| Cross-account transfer | Source and destination in different AWS accounts | Requires resource-based policies on the destination (e.g., bucket policy) trusting the source account's DataSync role |
| Bandwidth saturation | Nightly sync competes with production office traffic | Set an explicit bandwidth throttle on the task |

---

## 6. Troubleshooting

| Problem | Possible Cause | Solution |
|---|---|---|
| Agent offline | Network/firewall blocking outbound 443, or activation issue | Verify security group allows outbound HTTPS; re-check agent's connectivity to the DataSync endpoint |
| Slow transfer | Bandwidth or storage bottleneck | Check network throughput, remove/raise bandwidth throttle, check source/destination IOPS |
| Permission denied | IAM role or storage permissions | Review the auto-created DataSync IAM role, bucket policy, and NFS export permissions |
| Task failed | Invalid configuration or lost connectivity mid-run | Check CloudWatch Logs for the specific execution; verify location settings |
| Files missing at destination | Filters excluding them, or incremental sync skipped unchanged files (expected) | Verify include/exclude filters; confirm files actually changed since last run |
| "Get key" fails during activation | Security group blocking inbound TCP 80 from your browser, or agent still booting | Wait 2–3 minutes after launch; confirm the inbound rule allows your IP on port 80; alternatively get the key from the agent's local console or via `curl` and enter it manually in the console |

---

## 7. Interview Questions

### Beginner
1. What is AWS DataSync and what problem does it solve?
2. Why would you use DataSync instead of `rsync`?
3. What is a DataSync Agent, and when is it required?
4. What is the difference between a Location, a Task, and a Task Execution?
5. Which AWS storage services does DataSync support as a destination?

### Intermediate
6. How does DataSync's incremental synchronization actually work under the hood?
7. How is data integrity verified after a transfer?
8. Can DataSync preserve file permissions and timestamps? How?
9. What port does the Agent use, and does it need any inbound access?
10. How do you throttle DataSync's bandwidth usage?

### Advanced
11. When would you choose DataSync over AWS Snowball for a migration?
12. How would you migrate 500 TB from an on-prem NAS to S3 with minimal downtime?
13. How do you configure DataSync for a cross-account transfer?
14. How would you troubleshoot a DataSync job that suddenly became very slow?
15. How would you scale DataSync throughput using multiple agents, and what's the constraint on doing so?

---

## 8. Cheat Sheet

```
AGENT REQUIRED?
  On-prem NFS/SMB → S3/EFS/FSx  → YES
  S3 → S3 / EFS → EFS / EFS → S3 → NO

AGENT PORT
  Normal operation: outbound TCP 443 only
  Activation (temporary): inbound TCP 80 from your browser/client — auto-closed after activation

TASK FLOW
  Create Agent → Activate → Create Source Location →
  Create Destination Location → Create Task → Run → Monitor

TRANSFER MODES
  Initial  = full copy
  Incremental = only changed files

VERIFY OPTIONS
  Verify only transferred data (fast)
  Verify all data (thorough, slower)

ENCRYPTION
  In transit: TLS
  At rest: SSE-S3 / SSE-KMS (S3), KMS (FSx), encryption at rest (EFS)

MULTI-AGENT RULE
  Up to 8 agents per NFS/SMB/HDFS location, to scale throughput
```

---

## 9. Mastery Checklist

- [ ] Can explain what DataSync is and the problem it solves vs. manual scripts
- [ ] Know when an Agent is required vs. not required
- [ ] Can deploy and activate a DataSync Agent on EC2
- [ ] Can create NFS and S3 locations
- [ ] Can create and run a task, and read a task execution's logs/metrics
- [ ] Can prove incremental sync works (only changed files transfer)
- [ ] Can configure a recurring schedule
- [ ] Can explain data integrity verification and encryption layers
- [ ] Can compare DataSync vs rsync vs Snowball vs Storage Gateway
- [ ] Can troubleshoot a failed/slow task using CloudWatch Logs
- [ ] Can explain the multi-agent throughput scaling constraint
- [ ] Comfortable answering all beginner/intermediate/advanced interview questions above

---

## 10. Cleanup (Dependency-Ordered)

> Delete in this exact order to avoid dependency errors.

1. **DataSync Console → Tasks** → select the task → **Delete**
2. **DataSync Console → Locations** → delete the **NFS location**, then the **S3 location**
3. **DataSync Console → Agents** → select `nfs-migration-agent` → **Delete agent** (deactivates it)
4. **EC2 Console** → terminate the `datasync-agent` instance
5. **EC2 Console** → terminate the `nfs-server` instance
6. **S3 Console** → empty the bucket `brightwave-engineering-docs-<your-account-id>` (delete all object versions since versioning is enabled), then delete the bucket
7. **KMS Console** → if you created a dedicated customer-managed key for this lab, schedule it for deletion
8. **CloudWatch Logs** → delete the log group created for the DataSync task
9. **VPC/Security Group** → delete `datasync-lab-sg` if it was created solely for this lab

---
