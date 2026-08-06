# AWS DataSync 

**Scenario recap:** BrightWave Engineering Pvt Ltd wants to migrate an on-premises NFS file share to Amazon S3 using AWS DataSync, with nightly incremental syncs going forward. Since we don't have real on-prem hardware, this lab **simulates the on-prem NFS server using an EC2 instance**, and deploys the **DataSync Agent** as a second EC2 instance — entirely inside AWS.

> **Important:** AWS explicitly does **not recommend** deploying the Agent on EC2 when the real source is genuinely on-premises, because of the added network latency of routing on-prem traffic through the cloud first. In production, you'd deploy the Agent as a VMware/KVM/Hyper-V VM physically inside your datacenter, as close to the NFS/SMB server as possible. The EC2-based NFS server + EC2-based Agent setup below is purely a **lab simulation** so you can practice the workflow without physical hardware.

> All steps use the **AWS Management Console** (plus CloudShell/CLI only where the console requires it, e.g., looking up the Agent AMI ID). Region used in this lab: **ap-south-1 (Mumbai)**.

---

## Step 1 — Create the VPC networking (if not already present)

1. Go to **VPC Console → Your VPCs** and confirm you have a VPC with at least one **public subnet** (both EC2 instances will sit here for lab simplicity).
2. Note the VPC ID and Subnet ID — you'll need them for both EC2 instances.
3. Go to **Security Groups → Create security group**, name it `datasync-lab-sg`, and add these inbound rules:
   - SSH (22) from your IP
   - NFS (2049, TCP+UDP) from the security group itself (self-referencing, so agent ↔ NFS server can talk)
   - All traffic from the security group itself (self-referencing) — simplifies the lab

---

## Step 2 — Launch the simulated on-prem NFS server (EC2)

1. **EC2 Console → Launch Instance**
2. Name: `nfs-server`
3. AMI: **Amazon Linux 2023**
4. Instance type: `t3.medium`
5. Key pair: select or create one
6. Network: your lab VPC, public subnet, **Auto-assign public IP: Enable**
7. Security group: `datasync-lab-sg`
8. Storage: 20 GB gp3 (this simulates the NFS-exported volume)
9. Launch the instance and wait for it to reach **Running**

---

## Step 3 — Configure the NFS export

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

---

## Step 4 — Get the DataSync Agent AMI ID

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

---

## Step 5 — Launch the Agent EC2 Instance

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

---

## Step 6 — Activate the Agent

1. **DataSync Console → Agents → Create agent**
2. Under **Service endpoint**, choose **Public service endpoints in your current AWS Region** (simplest option for this lab; VPC endpoints are used for private-only setups)
3. Under **Activation key**, choose **Automatically get the activation key from your agent**
4. For **Agent address**, enter the `datasync-agent` instance's public IP, then click **Get key**
   - This works because your browser reaches the agent on port 80, which you opened in Step 5.
5. Agent name: `nfs-migration-agent`
6. Click **Create agent**

> Immediately after creation the agent may briefly show **OFFLINE** — this is expected and it should move to **ONLINE** within a minute or two.

> **If activation fails:** confirm your browser can reach the agent's public IP on port 80 (check the security group rule), or use the alternative "manually enter activation key" option by retrieving the key from the agent's local console instead.

---

## Step 7 — Create the Source Location (NFS)

1. **DataSync Console → Locations → Create location**
2. Location type: **NFS**
3. Agent: select `nfs-migration-agent`
4. NFS server: the **private IP** of `nfs-server`
5. Mount path: `/export/engineering-docs`
6. Click **Create location**

---

## Step 8 — Create the Destination Location (S3)

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

---

## Step 9 — Create the Task

1. **DataSync Console → Tasks → Create task**
2. Source location: the NFS location from Step 7
3. Destination location: the S3 location from Step 8
4. **Settings:**
   - Verify data: **Verify only data transferred**
   - Metadata: preserve ownership, permissions, timestamps
   - Filters: none (transfer everything for this lab)
   - Schedule: **Run on demand** for the first test run
   - Task logging: send logs to CloudWatch Logs (create/select a log group)
5. Review and **Create task**

---

## Step 10 — Run the Initial Transfer

1. Open the task, click **Start → Start with defaults**
2. Watch the **Task Execution** detail page — it shows files transferred, bytes transferred, duration, and throughput in real time.
3. Once status shows **SUCCESS**, verify in the S3 console that all 20 sample files landed under `/nfs-migration/`.

---

## Step 11 — Test Incremental Sync

1. Back on `nfs-server`, add a new file and modify an existing one:
   ```
   sudo bash -c 'echo "New revision" > /export/engineering-docs/drawing-1.txt'
   sudo bash -c 'echo "Brand new file" > /export/engineering-docs/drawing-21.txt'
   ```
2. Run the task again from the console.
3. Confirm the execution log shows only **2 files transferred**, not all 21 — this proves incremental sync is working.

---

## Step 12 — Schedule Nightly Syncs

1. Open the task → **Edit**
2. Under **Schedule**, choose a recurring schedule (e.g., cron `0 2 * * ? *` for 2:00 AM daily)
3. Save — the task will now run automatically every night.

---

## Step 13 — Verify Monitoring & Audit

1. **CloudWatch → Log groups** → open the DataSync log group → confirm per-file transfer logs appear.
2. **CloudWatch → Metrics → DataSync** → check `BytesTransferred`, `FilesTransferred`, `FilesVerified`.
3. **CloudTrail → Event history** → filter by event source `datasync.amazonaws.com` → confirm `CreateTask`, `StartTaskExecution` events are logged.

---
