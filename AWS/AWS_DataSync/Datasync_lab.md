# 🧪 AWS Lab: On-Prem (EC2-Simulated) → S3 Migration using AWS DataSync

---

## 📌 Scenario

You are a DevOps/Cloud Engineer at a company that has years of file data sitting on an **on-premises NFS file server**. Leadership wants this data migrated to **Amazon S3** for durability, cost savings, and to unlock downstream analytics/backup workflows — but the migration must be:

- **Non-disruptive** (business still writes files to the on-prem server during migration)
- **Incremental** (only new/changed files should re-transfer, not the whole dataset every time)
- **Verified** (data integrity must be provable, not assumed)

Since a real on-prem data center isn't available for this lab, we simulate "on-prem" using an **EC2 instance running an NFS server**. The lab uses **AWS DataSync** — the AWS-native data transfer service purpose-built for this exact use case — to move data from the NFS share into S3.

By the end of this lab, you will understand:
- How DataSync Agents bridge on-prem (or EC2-simulated on-prem) storage to AWS
- How NFS locations and S3 locations are defined
- How DataSync performs **incremental, checksum-verified transfers**
- How to correctly scope security groups for an agent-based transfer (a common real-world misconfiguration point)
- How to troubleshoot the most common real-world DataSync failures

---

## 🏗️ Architecture Diagram

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/c3c41880-893f-4de8-b3e6-94f10341f327" />


**Key architectural point:** DataSync doesn't move data through your laptop or the console. The **Agent** (running as an EC2 instance here) reads data directly off the NFS share and pushes it to S3 over HTTPS. The console only orchestrates — it is not in the data path.

**Key security point:** All the traffic the agent initiates (to the NFS server on 2049, and to AWS on 443) is **outbound from the agent** — so the agent's own security group barely needs any inbound rules at all. The only inbound rule the agent needs is port 80, and only temporarily, for activation.

---

## ✅ Prerequisites

| Requirement | Detail |
|---|---|
| AWS Account | With permissions for EC2, S3, DataSync, IAM |
| Region | Pick one region and use it for everything (e.g. `ap-south-1`) |
| VPC | Default VPC is fine, must have a public subnet |
| Budget awareness | EC2 (t2.micro + m5.large), S3 storage, DataSync per-GB fee all incur cost — see Cost Notes section |
| SSH client | Terminal / PuTTY to connect to the "on-prem" EC2 |

> ⚠️ **Cost note:** The DataSync Agent EC2 instance type recommended by AWS is **m5.large**, which is **not** Free Tier eligible. Budget for a few cents/hour while the agent is running. Terminate it as soon as the lab is done (see Cleanup section).

---

## 🧠 Concept Deep Dive

### What is AWS DataSync?
A **managed data transfer service** that moves large amounts of data between on-premises storage (NFS, SMB, HDFS, object storage) and AWS storage services (S3, EFS, FSx) — or between AWS storage services themselves. It's built to replace slow, error-prone `rsync`/`scp` scripts with something that handles parallelism, retries, encryption, scheduling, and verification natively.

### Why not just `aws s3 sync`?
`aws s3 sync` works file-by-file over a single thread from wherever you run it, has no agent-based parallelism, no built-in bandwidth throttling, and no data-integrity verification report. DataSync is designed for **TB-to-PB scale** migrations with production-grade guarantees.

### The DataSync Agent
The Agent is a **purpose-built virtual machine** (deployed here as an EC2 instance, but in real on-prem scenarios it's a VMware/Hyper-V/KVM OVA or a physical appliance) that:
- Mounts your on-prem storage (NFS/SMB) locally
- Talks to the DataSync service control plane over **HTTPS (443, outbound)**
- Streams data directly to AWS — the agent is the actual data mover, not the console

### Locations
DataSync uses the concept of **Locations** — reusable pointers to either a source or destination:
- **NFS Location** → server IP/hostname + export path + which Agent to use
- **S3 Location** → bucket + optional prefix/folder + IAM role for write access

### Transfer Modes
| Mode | Behavior |
|---|---|
| **Changed files only** (default, recommended) | Compares source vs destination metadata; transfers only new/modified files |
| **All data** | Re-transfers everything every run, ignoring destination state |

### Verification Modes
| Mode | Behavior |
|---|---|
| **Verify only the data transferred** | Checksums only the files moved in this run (fast, recommended for incremental syncs) |
| **Verify all data in transfer** | Checksums entire source and destination dataset every run (slow, thorough — use for final cutover) |
| **No verification** | Not recommended — skips integrity checks |

### How does DataSync know what changed? (the interview-favorite detail)
DataSync compares **file metadata** — size, modification timestamp, and (depending on verification mode) **checksum** — between source and destination. If metadata matches, the file is skipped. If it differs or the file is new, it's queued for transfer and checksummed after landing in S3 to confirm byte-for-byte integrity.

---

## 🧭 PHASE 1 — Create the S3 Bucket (Destination)

1. Open the **AWS Console** → search **S3**
2. Click **Create bucket**
3. Fill in:
   - **Bucket name:** `datasync-demo-bucket-<yourname>` (must be globally unique — add initials/numbers if taken)
   - **Region:** the same region you'll use for everything (e.g. `ap-south-1` — Mumbai)
4. Leave all other settings at default (Block Public Access **ON**, versioning off)
5. Click **Create bucket**

✅ **Checkpoint:** Bucket appears in your S3 bucket list with 0 objects.

---

## 🧭 PHASE 2 — Create Security Groups (do this before launching any instance)

Creating both security groups **now, in the right order**, avoids having to edit rules later. `sg-nfs-server` needs to reference `sg-datasync-agent` as a traffic source, so the agent's SG must exist first.

### Step 2a — Create `sg-datasync-agent` (no dependencies, create first)
1. **EC2 console → Security Groups → Create security group**
2. **Name:** `sg-datasync-agent`
3. **VPC:** your lab VPC (same one you'll use for everything)
4. **Inbound rules:**

   | Type | Port | Source | Purpose |
   |---|---|---|---|
   | HTTP | 80 | My IP | One-time agent activation only — AWS auto-closes this after activation completes |

5. **Outbound rules:** leave the default **Allow all traffic** rule in place. Do not remove it — the agent needs outbound access to AWS's DataSync service endpoints (port 443) and outbound access to the NFS server (port 2049), and both are covered by the default outbound-allow-all rule.
6. Click **Create security group**

> Why no inbound 443 or 2049 here: the agent only ever *initiates* those connections — it calls out to AWS and calls out to the NFS server. Neither requires an inbound rule on the agent's own SG.

### Step 2b — Create `sg-nfs-server` (references the agent's SG)
1. **Create security group** again
2. **Name:** `sg-nfs-server`
3. **VPC:** same VPC
4. **Inbound rules:**

   | Type | Port | Source | Purpose |
   |---|---|---|---|
   | SSH | 22 | My IP | So you can SSH in to configure NFS |
   | Custom TCP | 2049 | **`sg-datasync-agent`** (select the security group itself as the source, not an IP range) | Lets the DataSync agent mount the NFS export — scoped to only the agent, not the whole internet |

5. **Outbound rules:** leave default **Allow all traffic**.
6. Click **Create security group**

✅ **Checkpoint:** Two security groups exist: `sg-datasync-agent` (80 in) and `sg-nfs-server` (22 in, 2049 in from `sg-datasync-agent`). Nothing needs to be revisited later.

> 🔒 **Why this matters:** The commonly-seen shortcut of opening port 2049 to `0.0.0.0/0` (Anywhere) exposes your file server's data port to the entire internet. Scoping the source to a security group ID instead means only instances that are members of `sg-datasync-agent` — i.e., only your actual DataSync agent — can ever reach port 2049, regardless of what IP that instance happens to have.

---

## 🧭 PHASE 3 — Launch EC2 (Simulated "On-Prem" NFS Server)

1. Go to **EC2 → Instances → Launch instance**
2. Fill in:
   - **Name:** `datasync-nfs-server`
   - **AMI:** Ubuntu Server 22.04 LTS
   - **Instance type:** `t2.micro` (Free Tier eligible)
   - **Key pair:** select an existing key pair or create a new one and download the `.pem`
3. **Network settings:**
   - **VPC/Subnet:** same VPC used for the security groups; a public subnet (needs a public IP so you can SSH in)
   - **Auto-assign public IP:** Enable
   - **Firewall (security groups):** choose **Select existing security group** → pick **`sg-nfs-server`**
4. Click **Launch instance**
5. Note the **Public IPv4 address** and **Private IPv4 address** — you'll need both later.

✅ **Checkpoint:** Instance state = Running, status checks = 2/2 passed.

---

## 🧭 PHASE 4 — Connect to EC2

```bash
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@<PUBLIC-IP>
```

✅ **Checkpoint:** You have a shell prompt like `ubuntu@ip-172-31-x-x:~$`.

---

## 🧭 PHASE 5 — Set Up the NFS Server

### Install NFS server software
```bash
sudo apt update
sudo apt install -y nfs-kernel-server
```

### Create the shared directory
```bash
sudo mkdir -p /data/shared
sudo chmod 777 /data/shared
```
> Note: `chmod 777` is used here purely for lab simplicity so DataSync's default mount can read/write without UID/GID mapping issues. In production, scope permissions tightly and use `no_root_squash`/`root_squash` deliberately based on your security requirements.

### Add sample files
```bash
echo "Hello DataSync" > /data/shared/file1.txt
dd if=/dev/zero of=/data/shared/bigfile.bin bs=1M count=50
```
This creates a small text file and a 50 MB binary file so you can see DataSync handle both trivial and non-trivial transfer sizes.

### Configure the NFS export
```bash
sudo nano /etc/exports
```
Paste this line at the end of the file:
```
/data/shared *(rw,sync,no_subtree_check,no_root_squash)
```

**What each flag means:**
| Flag | Meaning |
|---|---|
| `rw` | Read-write access |
| `sync` | Writes are confirmed to disk before acknowledging — safer, matches real production NFS behavior |
| `no_subtree_check` | Disables subtree checking (improves reliability for exported subdirectories) |
| `no_root_squash` | Lets the DataSync agent (connecting as root) read files without UID remapping — needed for a clean lab. In production, evaluate this carefully. |

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

### Apply the export
```bash
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

### Verify the export
```bash
sudo exportfs -v
```
You should see output confirming `/data/shared` is exported with the flags above.

### (Optional but recommended) Confirm the daemon is only listening where expected
```bash
sudo ss -tlnp | grep 2049
```
This confirms NFSv4 is bound to TCP port 2049 — the only port you need open in the security group (Ubuntu 22.04's `nfs-kernel-server` negotiates NFSv4 by default, which consolidates the legacy NFSv3 `portmapper`/`mountd`/`nfsd` ports down to just 2049).

✅ **Checkpoint:** EC2 is now functioning as your "on-prem" NFS file server with 2 test files.

---

## 🧭 PHASE 6 — Launch the DataSync Agent

1. Go to **AWS Console → DataSync**
2. Left sidebar → **Agents → Create agent**
3. **Deployment/Activation:**
   - Choose **Amazon EC2** as the hosting platform (deploys the agent as an EC2 instance running AWS's DataSync AMI)
4. Fill in:
   - **Instance type:** `m5.large` (AWS-recommended minimum for the agent AMI — smaller types aren't offered because the agent needs the memory/network throughput headroom)
   - **VPC:** same VPC as your NFS EC2 instance
   - **Subnet:** the **public subnet** (the agent needs outbound internet access to reach the DataSync public service endpoint, and needs a public IP so the console can reach it on port 80 for activation)
   - **Auto-assign public IP:** Enable
   - **Security group:** select the existing **`sg-datasync-agent`** (created in Phase 2 — do not create a new one here)
5. Click **Launch agent** (or **Next**, depending on console flow — this provisions the EC2-based agent instance)

✅ **Checkpoint:** A new EC2 instance appears (agent), and the DataSync console shows an agent in "Not activated" state. No security group edits are needed — everything was already scoped correctly in Phase 2.

---

## 🧭 PHASE 7 — Activate the Agent

1. Note the **Public IP** of the newly launched agent EC2 instance
2. In a browser, go to:
   ```
   http://<agent-public-ip>
   ```
   This opens the agent's local activation web UI (served on port 80 directly from the agent instance).
3. The page auto-generates an **Activation Key**. Copy it.
4. Go back to the **DataSync console → Agents → Activate agent**
5. Paste the activation key, give the agent a friendly name (e.g. `nfs-onprem-agent`)
6. Click **Activate**

✅ **Checkpoint:** Agent status in the console changes to **ONLINE**. AWS automatically closes off reliance on port 80 once activation completes — you can remove that inbound rule from `sg-datasync-agent` afterward if you want to tighten things further, though it's not required for the rest of this lab.

> **If the activation page won't load:** it's almost always a security group issue — confirm port 80 is open from wherever your browser is connecting from (check "My IP" hasn't changed if you're on a different network than when the SG was created), and that the agent has a public IP with a route to the internet.

---

## 🧭 PHASE 8 — Create the Source Location (NFS)

1. DataSync console → **Locations → Create location**
2. Location type: **NFS**
3. Fill in:
   - **NFS server:** the **private IP** of your "on-prem" EC2 (the agent talks to it inside the VPC — private IP is correct and more secure than routing over the public IP)
   - **Mount path:** `/data/shared`
   - **Agents:** select the agent you activated in Phase 7
4. Click **Create location**

✅ **Checkpoint:** Location status = **Available**. If this fails, it's almost always `sg-nfs-server` not allowing port 2049 from `sg-datasync-agent` — double check Phase 2b.

---

## 🧭 PHASE 9 — Create the Destination Location (S3)

1. **Create location → Amazon S3**
2. Fill in:
   - **S3 bucket:** select `datasync-demo-bucket-<yourname>`
   - **Folder (prefix):** `/output/`
   - **IAM role:** choose **Autogenerate** (DataSync creates a scoped role with exactly the S3 permissions it needs — `s3:GetObject`, `s3:PutObject`, `s3:ListBucket`, etc., on this bucket only)
3. **Storage class:** Standard (default) is fine for this lab
4. Click **Create location**

✅ **Checkpoint:** Location status = **Available**.

---

## 🧭 PHASE 10 — Create the Task

1. **Tasks → Create task**
2. **Source location:** select your NFS location
3. **Destination location:** select your S3 location
4. Click **Next**

### Additional settings (important — this is where behavior is actually defined)
| Setting | Value | Why |
|---|---|---|
| **Transfer mode** | Changed files only | Enables true incremental sync — this is the whole point of the lab |
| **Verify mode** | Verify only data transferred | Fast, still gives checksum-level integrity confidence on what actually moved |
| **Overwrite behavior** | Always | Ensures the destination reflects the latest source state even if a file was manually edited in S3 |
| **Task logging (optional)** | Enable → send to CloudWatch Logs | Strongly recommended so you can see per-file transfer decisions |

5. Click **Create task**

✅ **Checkpoint:** Task status = **Available** (not yet running).

---

## 🧭 PHASE 11 — Run the Task

1. Select your task → click **Start**
2. Choose **Start with defaults**
3. Watch the execution detail page:
   - **Status** → Launching → Preparing → Transferring → Verifying → Success
   - Live metrics: **files transferred**, **data transferred**, **throughput**
4. Wait until:
   - **Status = SUCCESS**

✅ **Checkpoint:** Task execution shows 2 files transferred, ~50 MB transferred total.

---

## 🧭 PHASE 12 — Verify in S3

1. Go to **S3 → your bucket**
2. Navigate into `/output/`
3. Confirm you see:
   - `file1.txt`
   - `bigfile.bin`
4. Optionally download `file1.txt` and confirm its contents read `Hello DataSync`.

✅ **Checkpoint:** Files present in S3, byte sizes match the source.

---

## 🧭 PHASE 13 — Test Incremental Sync (the core DataSync value proposition)

1. Back on the "on-prem" EC2:
   ```bash
   echo "Second file" > /data/shared/file2.txt
   ```
2. In the DataSync console, select the same task → **Start** again
3. Watch the execution:
   - 👉 **Only `file2.txt` transfers.** `file1.txt` and `bigfile.bin` are skipped because their metadata is unchanged since the last successful sync.

✅ **Checkpoint:** Task execution shows **1 file transferred**, not 3 — this proves changed-files-only mode is working.

**Bonus test (optional):** Modify `file1.txt` (`echo "Updated" >> /data/shared/file1.txt`) and re-run — you should see exactly 1 file transferred again, confirming DataSync detects *modifications*, not just *new* files.

---

## 🧠 What Just Happened (Plain-English Recap)

1. DataSync's agent connected to your NFS export and enumerated files with their metadata (size, mtime).
2. On the first run, it compared that metadata against an empty S3 destination → everything was "new" → full transfer.
3. Data was streamed agent → AWS over HTTPS, landed in S3, and checksummed against the source to confirm integrity.
4. On the second run, it re-scanned the source, compared metadata against what's now in S3, and found only `file2.txt` didn't have a matching destination object → transferred just that one file.
5. This metadata-diff-then-checksum-verify pattern is what makes DataSync suitable for **ongoing, scheduled, production-grade sync** rather than a one-time copy tool.

---

## 🔒 Security Group Summary (quick reference)

| Security Group | Attached To | Inbound Rules | Outbound Rules |
|---|---|---|---|
| `sg-datasync-agent` | DataSync Agent EC2 | TCP 80 from My IP (activation only, temporary) | Default: allow all (needed for outbound 443 to AWS, outbound 2049 to NFS server) |
| `sg-nfs-server` | "On-prem" NFS EC2 | TCP 22 from My IP (SSH); TCP 2049 from `sg-datasync-agent` | Default: allow all |

Notice what's **not** here: no inbound 443 anywhere, no inbound 2049 on the agent's own SG, and no `0.0.0.0/0` source on the NFS port. Every rule maps to traffic that's actually initiated in that direction.

---

## ⚠️ Troubleshooting Reference

| Symptom | Likely Cause | Fix |
|---|---|---|
| NFS location creation fails / agent can't mount | `sg-nfs-server` not allowing port 2049 from `sg-datasync-agent` | Edit `sg-nfs-server` inbound rules, confirm the 2049 rule's source is the agent's security group ID, not an IP |
| Agent stuck "Not activated" | Port 80 blocked on `sg-datasync-agent`, or no internet route | Check inbound 80 from your current IP (it changes across networks); confirm the agent's subnet has a route to an Internet Gateway |
| Agent activation page won't load in browser | Agent has no public IP, or your IP isn't the one allowed in the SG | Confirm "Auto-assign public IP" was enabled at launch; re-check "My IP" hasn't changed since the SG rule was created |
| Task runs but S3 stays empty | Wrong mount path, or NFS export permissions too restrictive | Confirm mount path is exactly `/data/shared`; check `exportfs -v` output; check `chmod` on the directory |
| Task fails with permission error on S3 side | Autogenerated IAM role wasn't granted, or bucket policy conflicts | Recreate the S3 location and let DataSync autogenerate the role again; check bucket policy for explicit denies |
| Transfer very slow | Undersized agent instance, or bandwidth throttle set on the task | Confirm `m5.large` (not smaller); check task's bandwidth limit setting under Additional settings |
| Second run transfers ALL files again, not just the new one | Transfer mode was set to "All data" instead of "Changed files only" | Edit the task, confirm Transfer mode = Changed files only |
| `exportfs -v` shows nothing | `/etc/exports` syntax error or `exportfs -a` not run after edit | Re-check the exact line in `/etc/exports`, re-run `sudo exportfs -a` |

---

## 💰 Cost Notes

| Resource | Approx. cost driver |
|---|---|
| NFS EC2 (`t2.micro`) | Free Tier eligible (750 hrs/month if eligible account) |
| DataSync Agent EC2 (`m5.large`) | **Not** Free Tier — billed hourly while running, regardless of whether a task is actively syncing |
| DataSync transfer fee | Per-GB charged for data moved by DataSync (separate from EC2/S3 costs) |
| S3 storage | Standard storage rates for what lands in `/output/` |

The single biggest avoidable cost in this lab is leaving the **m5.large agent instance running** after you're done — it does nothing useful once you've validated the sync, so terminate it promptly.

---

## 🎯 Interview Q&A

**Q: How does DataSync know what changed between runs?**
A: It uses file **metadata** (size, modification time) to identify candidates, then verifies transferred data with **checksums** — so only new or modified files are moved, and integrity is provably confirmed rather than assumed.

**Q: Why does DataSync need an Agent instead of just using the S3 API directly from on-prem?**
A: The Agent handles the protocol translation (NFS/SMB → AWS APIs), parallelizes transfers, manages retries/backpressure, and gives you a single throttleable, monitorable component — none of which a plain `aws s3 sync` script provides at scale.

**Q: What's the difference between "Changed files only" and "All data" transfer modes?**
A: "Changed files only" performs a metadata diff and skips unchanged files — ideal for recurring/incremental syncs. "All data" re-transfers everything every run regardless of destination state — useful mainly for a guaranteed full refresh or first-time bulk copy validation.

**Q: Walk me through the security group design for a DataSync agent setup.**
A: The agent only needs one inbound rule — port 80, and only temporarily, for activation. Everything else the agent does (reaching AWS on 443, reaching the NFS server on 2049) is outbound traffic it initiates, which the default outbound-allow rule already covers. The NFS server's security group is the one that needs an inbound rule on 2049, and it should be scoped to the agent's security group specifically — not to a broad IP range — so only the agent can ever reach the file share's data port.

**Q: Why is the DataSync agent deployed in the public subnet in this lab?**
A: It needs a public IP and outbound connectivity so the console can reach it on port 80 for activation and so it can reach the DataSync service's public endpoints for ongoing sync traffic over HTTPS. In a stricter production setup, you'd instead deploy a **VPC interface endpoint for DataSync** and keep the agent fully private with no public IP at all.

**Q: How would you make this production-grade instead of lab-grade?**
A: Replace `no_root_squash`/`chmod 777` with least-privilege NFS export options, use a DataSync VPC endpoint instead of the public internet for agent-to-AWS traffic, remove the port-80 activation rule entirely once activation is complete, enable CloudWatch task logging, and schedule the task (cron-style) instead of manually clicking Start.

**Q: What's actually being checksummed, and when?**
A: Depending on verify mode, DataSync checksums either just the files transferred in that run or the entire source/destination dataset, comparing the computed checksum of what landed in S3 against the source file to confirm byte-for-byte fidelity — not just that a file with the same name and size exists.

---

## 📋 Cheat Sheet

```bash
# NFS server setup
sudo apt update && sudo apt install -y nfs-kernel-server
sudo mkdir -p /data/shared && sudo chmod 777 /data/shared
echo "/data/shared *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
sudo exportfs -v              # verify export
sudo ss -tlnp | grep 2049     # confirm NFSv4 listening on 2049 only

# Test files
echo "Hello DataSync" > /data/shared/file1.txt
dd if=/dev/zero of=/data/shared/bigfile.bin bs=1M count=50

# Incremental test
echo "Second file" > /data/shared/file2.txt
```

| Component | Value used in this lab |
|---|---|
| NFS export path | `/data/shared` |
| S3 destination prefix | `/output/` |
| Agent instance type | `m5.large` |
| NFS server instance type | `t2.micro` |
| Agent activation port | 80 (inbound, temporary) |
| Agent → AWS comms port | 443 (outbound) |
| Agent → NFS port | 2049 (outbound) |
| NFS server inbound port | 2049 (from `sg-datasync-agent` only) |
| Transfer mode | Changed files only |
| Verify mode | Verify only transferred data |
| Overwrite | Always |

---

## ✅ Mastery Checklist

- [ ] I can explain why DataSync uses an Agent instead of transferring directly via API
- [ ] I can explain the difference between NFS export flags: `rw`, `sync`, `no_subtree_check`, `no_root_squash`
- [ ] I can explain why the agent's security group needs almost no inbound rules (only port 80, temporarily)
- [ ] I correctly scoped the NFS server's security group to only allow port 2049 from the Agent's SG, not `0.0.0.0/0`
- [ ] I can explain the activation flow (port 80, one-time) versus ongoing sync traffic (port 443, outbound)
- [ ] I ran a first sync (full transfer) and a second sync (incremental — only new file moved)
- [ ] I can explain "Changed files only" vs "All data" transfer modes
- [ ] I can explain the two verify modes and when you'd choose each
- [ ] I know how DataSync detects changes (metadata diff + checksum verification)
- [ ] I completed cleanup and confirmed no lingering billable resources

---

## 💣 Cleanup (DO THIS — DataSync Agent + EC2 will keep billing otherwise)

Perform in this order — each step depends on the one before it being done first:

1. **Delete the DataSync Task**
   DataSync console → Tasks → select task → Delete
2. **Delete the DataSync Locations** (NFS and S3)
   DataSync console → Locations → select each → Delete
3. **Delete the DataSync Agent** (this deregisters it from the service — the underlying EC2 instance still needs separate termination in the next step)
   DataSync console → Agents → select agent → Delete
4. **Terminate both EC2 instances**
   - The "on-prem" NFS server (`datasync-nfs-server`)
   - The Agent instance (created automatically in Phase 6)
   EC2 console → Instances → select both → Instance state → Terminate
5. **Empty and delete the S3 bucket**
   S3 console → select bucket → Empty → confirm → then Delete bucket
6. **Delete the security groups** (`sg-datasync-agent`, `sg-nfs-server`) — only possible after the EC2 instances referencing them are fully terminated
7. **Delete the autogenerated IAM role** DataSync created for S3 access (IAM console → Roles → search for a role name containing `DataSync` or the bucket name)

✅ **Final checkpoint:** EC2 Instances list is empty of lab resources, S3 bucket is gone, DataSync console shows no tasks/locations/agents remaining, and both security groups are deleted.
