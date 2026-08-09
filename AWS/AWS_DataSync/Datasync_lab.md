# 🧪 AWS DataSync Lab (FINAL) — On-Prem (EC2-Simulated) NFS → S3
### Matches the current DataSync console wizard exactly — Aug 2026

---

## 🚨 Read this before you touch the console — corrects everything from last night

1. **Your region (`eu-north-1` / Stockholm) is NOT the problem.** DataSync has been fully supported there since 2019, including the EC2 agent. You do **not** need to switch to Mumbai or N. Virginia. Ignore that advice.
2. **There is no "Launch agent (EC2)" auto-launch button anymore.** The current console wizard shows Hypervisor = Amazon EC2 with a note: *"Create an EC2 instance using the AMIs provided in our User Guide."* That's the whole story — AWS expects you to grab the AMI ID yourself (one CLI command) and launch EC2 manually in a separate tab. This matches exactly what you saw.
3. **Instance size depends on Task mode**, which is now chosen *before* you pick the hypervisor:

| Task mode | When to use | Recommended EC2 instance |
|---|---|---|
| **Basic** | Simple lab/test transfers, dataset under 20M files | **`m5.2xlarge`** |
| **Enhanced** (AWS default recommendation) | Production, unlimited files, better metrics/logging | **`m6a.2xlarge`** |

For this lab (2 test files), **Basic mode + `m5.2xlarge`** is the cheaper, correct choice. Use Enhanced only if you specifically want to practice that mode.

4. Minimum instance size for either mode is **2xlarge** — never go smaller, the console will reject it.

---

## 📌 Scenario

Migrate files from a simulated on-prem **NFS server (EC2)** to **S3** using **AWS DataSync**, non-disruptively and incrementally, with checksum-verified integrity. Custom VPC built from scratch so you see every networking piece. The lab then extends to **two-way sync** using a second, reversed task.

---

## 🏗️ Architecture (current infrastructure — includes two-way sync)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/dbedbf90-5366-4932-87d2-13496f702b3a" />


**Key architectural points:**
- The **Agent** (EC2, `m5.2xlarge`, Basic mode) is the actual data mover for *both* tasks — it reads NFS for Task A and writes NFS for Task B. Same physical agent, reused.
- **Two independent one-way tasks**, not one bidirectional task — DataSync has no native two-way mode (see Appendix below).
- Tasks run on **offset schedules** (`:00/:15/:30/:45` vs `:07/:22/:37/:52`) so they don't race each other every cycle.
- S3 is reached via the Internet Gateway (or a VPC Gateway Endpoint in production) — it isn't "inside" the VPC.
- All traffic the agent initiates (NFS on 2049, AWS on 443) is outbound — the agent's own SG barely needs inbound rules; port 80 is only for the one-time activation handshake.

### Base one-way version (if you're not doing the two-way appendix)

```
[On-prem NFS EC2] <--2049 (private IP)-- [DataSync Agent EC2] --443 HTTPS--> [DataSync Service] --> [S3 Bucket]
   sg-nfs-server                              sg-datasync-agent
   (22 in, 2049 from agent SG)                (80 in, temporary — activation only)
```

The Agent is the actual data mover — it reads NFS directly and pushes to S3 over HTTPS. The console only orchestrates.

---

## ✅ Prerequisites

| Requirement | Detail |
|---|---|
| AWS Account | Permissions for VPC, EC2, S3, DataSync, IAM, SSM |
| Region | One region, anywhere DataSync is supported (yours — `eu-north-1` — is fine) |
| AWS CLI | Installed and configured (`aws configure`) — needed for the AMI lookup command. CloudShell also works if you don't want to install anything locally |
| Budget | EC2 (t2.micro + m5.2xlarge), S3 storage, DataSync per-GB fee |

> ⚠️ **Cost note:** `m5.2xlarge` is **not** Free Tier eligible. Terminate it the moment you're done testing.

---

## 🧠 Concept Deep Dive

**DataSync** = managed data transfer service moving data between on-prem storage (NFS/SMB/HDFS/object storage) and AWS storage (S3/EFS/FSx), or between AWS services. Replaces slow `rsync`/`scp` with parallelism, retries, encryption, scheduling, verification.

**Why not `aws s3 sync`?** Single-threaded, no agent parallelism, no bandwidth throttling, no integrity report. DataSync is built for TB–PB scale.

**The Agent:** A VM (here, EC2 running AWS's DataSync AMI) that mounts on-prem storage locally, talks to the DataSync control plane over HTTPS (443, outbound), and streams data to AWS.

**Task modes:**
| Mode | Behavior |
|---|---|
| Basic | Supports up to 50M files/folders total; lower performance; simpler resource footprint |
| Enhanced | No file quota; advanced metrics, more detailed logs, higher performance |

**Locations:** reusable pointers — NFS Location (server + export path + agent) or S3 Location (bucket + prefix + IAM role).

**Transfer modes:**
| Mode | Behavior |
|---|---|
| Changed files only (default) | Metadata diff; transfers only new/modified files |
| All data | Re-transfers everything every run |

**Verify modes:**
| Mode | Behavior |
|---|---|
| Verify only data transferred | Checksums only files moved this run |
| Verify all data in transfer | Checksums entire dataset every run |

**How DataSync detects changes:** Compares file metadata (size, mtime). Match → skip. Differ/new → transfer, then checksum after landing in S3 to confirm byte-for-byte integrity.

---

## 🧭 PHASE 1 — Build the VPC from Scratch

### 1a — Create the VPC
**VPC console → Create VPC → VPC only**
- Name: `datasync-lab-vpc` · IPv4 CIDR: `10.0.0.0/16` · No IPv6 · Tenancy: Default
- **Create VPC**

### 1b — Create a Public Subnet
**Subnets → Create subnet** → VPC: `datasync-lab-vpc`
- Name: `datasync-public-subnet` · AZ: any (e.g. `eu-north-1a`) · CIDR: `10.0.1.0/24`
- **Create subnet**

### 1c — Enable Auto-Assign Public IP
Select subnet → **Actions → Edit subnet settings** → check **Enable auto-assign public IPv4 address** → **Save**

### 1d — Internet Gateway
**Internet Gateways → Create internet gateway** → name `datasync-lab-igw`
Select it → **Actions → Attach to VPC** → `datasync-lab-vpc`

### 1e — Route Table
**Route Tables → Create route table** → name `datasync-public-rt`, VPC `datasync-lab-vpc`
- **Routes → Edit routes → Add route**: `0.0.0.0/0` → Target: Internet Gateway → `datasync-lab-igw` → **Save**
- **Subnet associations → Edit subnet associations** → check `datasync-public-subnet` → **Save**

✅ **Checkpoint:** IGW attached, route table has `0.0.0.0/0 → igw`, associated with the subnet.

---

## 🧭 PHASE 2 — Create the S3 Bucket

**S3 → Create bucket**
- Name: `datasync-demo-bucket-<yourname>` (globally unique) · Region: same as VPC
- Leave defaults → **Create bucket**

---

## 🧭 PHASE 3 — Create Security Groups (before launching anything)

### 3a — `sg-datasync-agent` (create first)
VPC: `datasync-lab-vpc`
- **Inbound:** HTTP (80) from **My IP** — temporary, for activation only
- **Outbound:** default Allow all

### 3b — `sg-nfs-server`
VPC: `datasync-lab-vpc`
- **Inbound:** SSH (22) from My IP · Custom TCP 2049 from **`sg-datasync-agent`** (select the SG as the source, not an IP)
- **Outbound:** default Allow all

✅ **Checkpoint:** Both SGs exist. 2049 scoped to the agent's SG only, never `0.0.0.0/0`.

---

## 🧭 PHASE 4 — Launch EC2 ("On-Prem" NFS Server)

**EC2 → Launch instance**
- Name: `datasync-nfs-server` · AMI: Ubuntu Server 22.04 LTS · Type: `t2.micro`
- Key pair: existing or new
- **Network settings (Edit):** VPC `datasync-lab-vpc` · Subnet `datasync-public-subnet` · Auto-assign public IP: Enable · Security group: select existing → `sg-nfs-server`
- **Launch instance**
- Note the **Public IPv4** and **Private IPv4**.

✅ **Checkpoint:** Running, 2/2 status checks passed.

---

## 🧭 PHASE 5 — Connect to EC2

```bash
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@<PUBLIC-IP>
```

If this hangs: revisit Phase 1e (route table) and Phase 3b (SSH rule).

---

## 🧭 PHASE 6 — Set Up the NFS Server

```bash
sudo apt update
sudo apt install -y nfs-kernel-server

sudo mkdir -p /data/shared
sudo chmod 777 /data/shared   # lab simplicity only

echo "Hello DataSync" > /data/shared/file1.txt
dd if=/dev/zero of=/data/shared/bigfile.bin bs=1M count=50

sudo nano /etc/exports
```
Add this line:
```
/data/shared *(rw,sync,no_subtree_check,no_root_squash)
```

| Flag | Meaning |
|---|---|
| `rw` | Read-write |
| `sync` | Writes confirmed to disk before ack |
| `no_subtree_check` | Disables subtree checking |
| `no_root_squash` | Lets the agent (as root) read files without UID remapping |

```bash
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
sudo exportfs -v                 # confirm export
sudo ss -tlnp | grep 2049        # confirm NFSv4 on 2049 only
```

✅ **Checkpoint:** EC2 is your "on-prem" NFS server with 2 test files.

---

## 🧭 PHASE 7 — Get the DataSync Agent AMI (this replaces the missing "Launch agent" button)

Open **CloudShell** (icon in the top AWS console bar) or your local terminal with AWS CLI configured. Run:

```bash
aws ssm get-parameter --name /aws/service/datasync/ami --region eu-north-1
```

Replace `eu-north-1` with your actual region if different.

- The output JSON has a `"Value"` field — that's your **AMI ID** (looks like `ami-0123456789abcdef0`). Copy it.
- If you get `ParameterNotFound`, try the versioned parameter instead:
  ```bash
  aws ssm get-parameter --name /aws/service/datasync/ami/v3 --region eu-north-1
  ```

✅ **Checkpoint:** You have a real AMI ID string copied somewhere safe.

---

## 🧭 PHASE 8 — Launch the Agent EC2 Instance (manual, using the AMI)

> ⚠️ **Don't use the "Browse more AMIs" modal to search your AMI ID.** That modal does a keyword/text search, not an exact ID lookup — searching an AMI ID there returns unrelated results (Amazon Linux, NVIDIA GPU AMIs, ECS-optimized images, etc.) because none of their names/descriptions contain your ID string. This wasted time in testing; use the method below instead, which is confirmed working.

### Confirmed-working method: direct launch URL
1. In a new tab, go to this exact URL (swap in your own AMI ID and region if different):
   ```
   https://us-east-1.console.aws.amazon.com/ec2/home?region=us-east-1#LaunchInstances:ami=ami-0f063e6b693a082c4
   ```
2. **Careful when typing/pasting this** — if the URL gets truncated (e.g. cut off at `region=us-east-` with no `1`), AWS will show a **"Region Unsupported"** error page that looks like a real regional-availability problem but is actually just a broken URL. Select the entire address bar and retype/repaste cleanly if this happens.
3. This opens the Launch Instance wizard with your AMI **already correctly selected** — confirmed working in testing. The Summary panel on the right should show:
   - **Software Image (AMI):** `aws-datasync-2.0.x-x86_64-gp2`, with your AMI ID underneath
   - It won't show a description/logo the way Marketplace AMIs do — that's normal, it's an Amazon-owned service AMI, not a Marketplace listing

### Finish the launch form
1. **Name:** `datasync-agent`
2. **Instance type:** change from the default (often `t3.micro`) to **`m5.2xlarge`** — must be at least 2xlarge, smaller types get rejected during activation. Ignore any "Free tier eligible" badge on smaller types; it doesn't apply here.
3. **Key pair:** not required (activation happens over HTTP, not SSH) — select "Proceed without a key pair" if prompted, or attach one anyway if you want it for later
4. **Network settings (Edit):**
   - VPC: `datasync-lab-vpc`
   - Subnet: `datasync-public-subnet`
   - Auto-assign public IP: **Enable**
   - Firewall (security groups): **Select existing security group** → `sg-datasync-agent` (the Summary panel will say "New security group" by default — you must change this)
5. **Storage:** leave at the default (80 GiB) — don't shrink it
6. Double-check the Summary panel: AMI ID correct, instance type `m5.2xlarge`, security group `sg-datasync-agent` — then **Launch instance**
7. Wait for **Running + 2/2 status checks passed**, then wait **3–5 more minutes** — status checks passing doesn't guarantee the agent's internal activation web server is live yet
8. Note the agent's **Public IPv4 address**

### Fallback: CLI method (use if the launch URL/wizard gives you trouble)
```bash
# Get your subnet ID and security group ID first:
aws ec2 describe-subnets --filters "Name=tag:Name,Values=datasync-public-subnet" --query "Subnets[0].SubnetId" --region us-east-1
aws ec2 describe-security-groups --filters "Name=group-name,Values=sg-datasync-agent" --query "SecurityGroups[0].GroupId" --region us-east-1

# Then launch:
aws ec2 run-instances \
  --image-id ami-0f063e6b693a082c4 \
  --instance-type m5.2xlarge \
  --subnet-id <your-subnet-id> \
  --security-group-ids <your-sg-id> \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=datasync-agent}]'
```

✅ **Checkpoint:** Agent EC2 instance running, public IP noted. This instance is standalone right now — DataSync doesn't know about it yet. That happens in the next phase.

---

## 🧭 PHASE 9 — Create the Agent in DataSync + Activate (matches your exact screen)

1. **DataSync console → Agents → Create agent**
2. **Source location:** Network File System (NFS)
3. **Destination location:** Amazon S3
4. **Task mode:** **Basic** (matches the `m5.2xlarge` instance you launched — if you chose Enhanced instead, you needed `m6a.2xlarge` in Phase 8)
5. **Hypervisor:** Amazon EC2
6. **Service endpoint:** Public service endpoint in your region (default is fine for this lab)
7. **Activation key** — choose ONE of these two paths:

### Path A — Automatic (try first)
- Select **Automatically get the activation key from your agent**
- **Agent address:** `http://<agent-public-ip>` (must include `http://`, and this is an IP — never paste it into the "Activation key" box, that's a different field expecting a code string like `ABCDE-12345-...`)
- Click **Get key**
- If it succeeds, the key auto-fills → give the agent a name (e.g. `nfs-onprem-agent`) → **Create agent**

### Path B — If "Get key" hangs or errors
1. Open a **new browser tab** → go directly to `http://<agent-public-ip>`
   - **Loads, shows an Activation Key** → copy it → back in DataSync console, select **Manually enter your agent's activation key** → paste the key (not the IP) → name the agent → **Create agent**. Done.
   - **Times out** → connectivity issue, go through this checklist:
     - Agent instance Running, 2/2 checks, **at least 3–5 min since launch**
     - Public IP re-copied fresh from EC2 console (changes if ever stopped/started)
     - `sg-datasync-agent` inbound rule is TCP **80**, source **My IP** — refresh this rule if your network changed since creating it
     - Route table from Phase 1e still associated with the subnet
<img width="3325" height="1703" alt="image" src="https://github.com/user-attachments/assets/64e3a566-c07f-44da-813a-82bd783f57df" />
<img width="3326" height="1557" alt="image" src="https://github.com/user-attachments/assets/f7a1848a-a8f6-40a6-b746-82e6910d0917" />
<img width="3332" height="1724" alt="image" src="https://github.com/user-attachments/assets/b63e314e-f589-4dde-a10d-c43c8b7ed557" />


✅ **Checkpoint:** Agent status in DataSync console = **ONLINE**. AWS auto-closes port 80 after activation.

---

## 🧭 PHASE 10 — Create the Task (this now builds both Locations for you, inline)

> ⚠️ **Console change from the old flow:** there used to be a separate **Locations** menu where you created the NFS location and S3 location first, then a separate **Tasks → Create task** step where you just picked from a dropdown. That's gone. The current console **merges all of it into one "Create task" wizard** with 4 steps: Configure source location → Configure destination location → Configure settings → Review. You still end up with the same NFS Location and S3 Location objects behind the scenes — you just create them inline as part of this one flow instead of as separate standalone steps.

**DataSync console → Tasks → Create task**

### Step 1 — Configure source location
- **Source location options:** leave on **Create a new location**
- **Location type:** Network File System (NFS) *(should already be selected)*
- **Region:** confirms it matches your VPC's region — DataSync requires NFS transfers to stay in the current Region
- **Agents:** select your activated agent from the dropdown (e.g. `My-Agent`) — it should show status **Basic mode** or **Enhanced mode** matching what you set up
- **NFS server:** enter the **private IPv4** of your "on-prem" EC2 (from EC2 console → `datasync-nfs-server` → Private IPv4 address — not the public one, the agent talks to it inside the VPC)
- **Mount path:** `/data/shared` (must exactly match the export path from Phase 6)
- Leave **Additional settings** collapsed/default
- Click **Next**

<img width="3310" height="1621" alt="image" src="https://github.com/user-attachments/assets/8fb3c634-6240-4a09-9d99-2d3ad258e143" />



✅ If this step fails on Next: almost always `sg-nfs-server` not allowing port 2049 from `sg-datasync-agent` — recheck Phase 3b.

### Step 2 — Configure destination location
- **Destination location options:** Create a new location
- **Location type:** Amazon S3
- **S3 bucket:** select `datasync-demo-bucket-<yourname>`
- **Folder (prefix):** `/output/`
- **IAM role:** Autogenerate (DataSync creates a scoped role with exactly the S3 permissions it needs)
- **Storage class:** Standard
- Click **Next**

<img width="3294" height="1709" alt="image" src="https://github.com/user-attachments/assets/379e85f1-69b3-48e8-ac5f-e39635354083" />
<img width="3306" height="1188" alt="image" src="https://github.com/user-attachments/assets/ed5ca104-4548-45fe-a387-42016100cfdb" />

<img width="3330" height="1372" alt="image" src="https://github.com/user-attachments/assets/c30b3dad-7079-4b22-8670-d784eb1a1531" />


### Step 3 — Configure settings

This screen exposes more granular controls than older versions of the console did. Go field by field:

| Field | What to select | Why / use case |
|---|---|---|
| **Task mode** | Should already show **Basic** (locked/greyed out) | It's forced to match your agent's mode — you set up a Basic-mode agent in Phase 8/9, so the task must be Basic too. **This can't be changed after the task is created**, so if it's showing Enhanced by mistake, fix it now before proceeding. |
| **Name** (optional) | `datasync-lab-task`, or leave blank | Cosmetic only — AWS auto-names it if you skip this. Useful once you have multiple tasks and need to tell them apart. |
| **Contents to scan** | Leave as **Everything** | You want the whole `/data/shared` folder transferred. Use **Excludes** only if you had subfolders/files you deliberately wanted DataSync to skip (e.g. temp files, logs) — not needed for this lab. |
| **Transfer mode** | **Transfer only data that has changed** | This *is* the incremental-sync behavior the whole lab exists to demonstrate — DataSync diffs file metadata and only moves what's new/modified. The alternative, "Transfer all data," re-copies everything every run and is only useful for a guaranteed full refresh or first-time bulk validation. |
| **Verification** | **Verify only transferred data** | Checksums just the files that moved in this run — fast, and still gives integrity confidence. Use "Verify all data" instead only for a final cutover/migration where you want to checksum the entire source and destination datasets every run (slower). |
| **Bandwidth limit** | **Use available** | No need to throttle for a 50 MB lab transfer. In production, you'd cap this if the sync competes with business traffic during work hours. |
| **Keep deleted files** | Leave **ON** (default — keep) | If a file gets deleted from `/data/shared`, this setting means DataSync leaves the already-synced copy in S3 alone rather than deleting it too. Safer default for a lab (and for most real backup/archival use cases) — you don't want DataSync silently deleting your S3 data because someone deleted the source file. |
| **Overwrite files** | Turn **ON** | When source data (including metadata) changes, this lets the updated version overwrite what's already in S3. You need this ON for Phase 13's incremental test to actually show `file1.txt` being re-transferred after you edit it — without it, DataSync would see a same-named object already exists and skip it. |
| **Copy ownership** | Leave **Yes** (default) | Preserves the source file's UID/GID (owner) metadata on the S3 object. Matters if you care about faithfully reproducing on-prem file ownership info in your migration — harmless to leave on for a lab. |
| **Copy permissions** | Leave **Yes** (default) | Preserves POSIX permission bits (rwx) as S3 object metadata. Same reasoning — faithful metadata copy, no downside for this lab. |
| **Copy timestamps** | Leave **Yes** (default) | Preserves the source file's modification time as S3 object metadata. This is actually important for the *change detection* itself — DataSync's "changed files only" comparison partly relies on timestamps matching, so this should stay on. |
| **Queueing** | Leave **Yes** (default) | Lets DataSync queue up multiple task executions if you try to start one while another is still running, instead of rejecting the second attempt. Sensible default, especially relevant once you add a Schedule. |
| **Schedule** | Choose based on your goal — see below | Two valid approaches: |

**Schedule — pick one:**
- **Not scheduled** (what the lab originally recommended): you control exactly when the sync runs via the **Start** button, so you can watch each Phase 11/13 execution manually and see cause → effect clearly. Best if you're learning the mechanics.
- **Rate expression, `rate(15 minutes)`** (or the cron equivalent `cron(0/15 * * * ? *)`): the task now runs automatically every 15 minutes in the background, including empty runs when nothing's changed. Good for practicing scheduling, but remember this keeps firing (and can incur small ongoing CloudWatch/API costs) until you either set it back to Not scheduled or delete the task — don't forget it's running after you've finished testing.

| **Task report** | Leave **None** | Optional detailed per-file report written to S3 (extra S3 pricing). Not needed for the lab — CloudWatch logging (next field) already gives you visibility into what transferred. |
| **Logging → Log level** | Change from the default **"Log basic information such as transfer errors"** to the **most detailed option available** (often labeled "Log all transfer operations" or similar) | The default only logs errors — you won't see which files were skipped vs transferred in CloudWatch. Bump this up if you want that visibility for Phase 13's incremental test. If you leave it on Basic (also a valid choice), you can still confirm file counts through the task execution UI itself instead of CloudWatch — you just won't get per-file logs. |
| **CloudWatch log group** | Leave **Autogenerate** (creates `/aws/datasync`) | DataSync creates and manages the log group for you — no need to pre-create one. |
| CloudWatch resource policy prompt (if shown) | **Accept/Allow** | DataSync needs this policy to actually be permitted to write logs to CloudWatch on your behalf. Required for logging to function at all. |

Click **Next**

### Step 4 — Review

Confirm everything on the summary page matches what you configured. A correct review page looks like this:

| Section | Field | Expected value |
|---|---|---|
| Source location | Type / Path / Server / Agent | NFS / `/data/shared` / your NFS server's private IP / your agent, Basic mode |
| Destination location | Type / Path / S3 bucket / Storage class / IAM role | S3 / `/output/` / your bucket name / Standard / autogenerated `AWSDataSyncS3BucketAccess-...` role |
| Settings | Transfer mode | Transfer only data that has changed |
| Settings | Verification | Verify only transferred data |
| Settings | Keep deleted files / Overwrite files | Yes / Yes |
| Settings | Copy ownership / permissions / timestamps / Queueing | Yes / Yes / Yes / Yes (all defaults) |
| Settings | Schedule | Either "Not scheduled" or your chosen rate/cron expression |
| Settings | Task report | None |
| Settings | Logging | Your chosen log level; log group `/aws/datasync` |

If anything doesn't match, click **Previous** to go back and fix the relevant step — settings can still be changed at this point, nothing is final until you click **Create task**.

Click **Create task**.

✅ **Checkpoint:** Task status = **Available** (or already running its first execution, if you scheduled it). Behind the scenes this created one NFS Location, one S3 Location, and one Task — all three now show up individually if you check the DataSync left nav.

---

## 🧭 PHASE 11 — Run the Task

Select task → **Start** → **Start with defaults**
Watch: Launching → Preparing → Transferring → Verifying → **Success**

✅ **Checkpoint:** 2 files transferred, ~50 MB total.

<img width="3299" height="1576" alt="image" src="https://github.com/user-attachments/assets/781df9cc-8655-418a-a809-a29a19f921e9" />

---

## 🧭 PHASE 12 — Verify in S3

**S3 → your bucket → /output/** — confirm `file1.txt` and `bigfile.bin` exist. Optionally download `file1.txt`, confirm it reads `Hello DataSync`.

<img width="3340" height="1418" alt="image" src="https://github.com/user-attachments/assets/6a356cef-d37c-460c-83b0-c437e9b97c95" />

---

## 🧭 PHASE 13 — Test Incremental Sync

```bash
echo "Second file" > /data/shared/file2.txt
```
Re-run the same task → only `file2.txt` transfers; the other two are skipped (metadata unchanged).

**Bonus:** `echo "Updated" >> /data/shared/file1.txt` → re-run → exactly 1 file transferred, confirming DataSync detects modifications, not just new files.


---



## ⚠️ Troubleshooting Reference (final, corrected)

| Symptom | Likely Cause | Fix |
|---|---|---|
| No "Launch agent (EC2)" button anywhere | **Not a bug or region issue.** Current console requires manual AMI lookup + manual EC2 launch | Follow Phase 7–8 exactly |
| Told to "switch regions" to fix this | **False.** DataSync + EC2 agent is fully supported in `eu-north-1` and virtually every commercial region | Ignore; stay in your region |
| Can't SSH into NFS EC2 | Route table not associated, or IGW not attached | Revisit Phase 1d/1e |
| EC2 has no public IP | Auto-assign not enabled | Revisit Phase 1c or enable at launch |
| `Value '<ip>' at 'activationKey' failed` | Typed an IP into the **Activation key** field instead of **Agent address** | Use the Agent address field for the IP; Activation key field only ever takes a code string |
| "Get key" hangs / times out | Agent not fully booted yet, or stale "My IP" SG rule | Wait 3–5 min after launch; refresh the My IP rule |
| Direct `http://<agent-ip>` browser test also fails | SG port 80 not actually open, or instance not Running yet | Recheck SG rule is TCP 80; recheck instance status |
| Console rejects your chosen instance type | Instance smaller than 2xlarge, or wrong size for the task mode you picked | Basic mode → `m5.2xlarge`; Enhanced mode → `m6a.2xlarge` |
| Searching your AMI ID in "Browse more AMIs" returns unrelated results (Amazon Linux, NVIDIA, ECS AMIs) | That search box does keyword/text matching on name & description, not exact AMI ID lookup | Skip it — use the direct launch URL: `https://<region>.console.aws.amazon.com/ec2/home?region=<region>#LaunchInstances:ami=<ami-id>` |
| Page shows "Region Unsupported" listing supported regions | The launch URL got truncated while typing/pasting (e.g. `region=us-east-` missing the `1`) — looks like a region problem but isn't | Retype/repaste the full URL carefully, or select the whole address bar first to avoid partial overwrite |
| Step 1 of Create task won't advance / errors on Next | `sg-nfs-server` not allowing 2049 from agent SG (this step is where the NFS location actually gets created now) | Fix Phase 3b |
| Task runs but S3 stays empty | Wrong mount path or export permissions | Confirm `/data/shared`; check `exportfs -v` |
| Second run transfers ALL files again | Transfer mode set to "All data" | Edit task → Changed files only |

---

## 💰 Cost Notes

| Resource | Driver |
|---|---|
| VPC/Subnet/IGW/Route Table | Free |
| NFS EC2 (`t2.micro`) | Free Tier eligible |
| Agent EC2 (`m5.2xlarge` Basic mode) | Not Free Tier — billed hourly while running |
| DataSync transfer fee | Per-GB moved |
| S3 storage | Standard rates |

Terminate the agent instance the moment you're done — it's the single biggest avoidable cost in this lab.

---




## 📋 Cheat Sheet

```bash
# NFS server setup
sudo apt update && sudo apt install -y nfs-kernel-server
sudo mkdir -p /data/shared && sudo chmod 777 /data/shared
echo "/data/shared *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
sudo exportfs -v
sudo ss -tlnp | grep 2049

# Test files
echo "Hello DataSync" > /data/shared/file1.txt
dd if=/dev/zero of=/data/shared/bigfile.bin bs=1M count=50
echo "Second file" > /data/shared/file2.txt

# Get the agent AMI (run this instead of looking for a "launch" button)
aws ssm get-parameter --name /aws/service/datasync/ami --region eu-north-1
```

| Component | Value |
|---|---|
| VPC CIDR | `10.0.0.0/16` |
| Subnet CIDR | `10.0.1.0/24` |
| NFS export path | `/data/shared` |
| S3 destination prefix | `/output/` |
| Agent instance type | `m5.2xlarge` (Basic mode, min 2xlarge required) |
| NFS server instance type | `t2.micro` |
| Agent activation port | 80 (inbound, temporary) |
| Agent → AWS port | 443 (outbound) |
| Agent → NFS port | 2049 (outbound) |
| Transfer mode | Changed files only |
| Verify mode | Verify only transferred data |

---


---

## 🔁 APPENDIX — Setting Up Two-Way Sync (NFS ↔ S3)

### Why this needs two tasks, not one

DataSync tasks are **one-directional** — a task has exactly one source and one destination, and it only ever pushes data that direction. There's no built-in "bidirectional" toggle. Your original task (NFS → S3) only reacts to changes made in `/data/shared` — if someone uploads a file directly to S3, DataSync never notices or touches it, because that file was never present in the *source* location the task is watching.

**True two-way sync = two separate one-way tasks pointed at each other:**
- Task A: NFS → S3 (already built earlier in this guide)
- Task B: S3 → NFS (new — mirror image of Task A)

### ⚠️ Trade-offs to understand before building this

| Risk | What it means | Mitigation |
|---|---|---|
| **No conflict resolution** | If the same file is edited on both NFS and S3 between sync runs, whichever task runs *last* wins — the other side's edit is silently overwritten. DataSync has no merge logic, no "conflict" flag, no warning. | Don't edit the same file from both sides in the same sync window. If you need real conflict-aware sync, DataSync isn't the right tool — look at something with versioning/merge logic instead. |
| **Sync "chatter"** | Task A writes a file to S3 → Task B sees it as new and pulls it to NFS, which can touch its timestamp → Task A sees it as "changed" again next run. Usually harmless (same content, small repeated transfers) but adds noise to your logs and a little extra transfer cost. | Stagger the two tasks' schedules so they don't run back-to-back constantly (see below). |
| **Deletion asymmetry** | With "Keep deleted files = Yes" on both tasks (the safe default), deleting a file on one side will **not** delete it on the other side on the next sync — the two locations can drift out of sync on deletions specifically, even though they stay in sync on additions/edits. | Understand this is by design for safety. If you actually want deletions to propagate, you'd change "Keep deleted files" to delete on both tasks — but that's riskier (a mistaken local delete becomes a permanent remote delete too). |

### What actually happens when the same file exists on both sides

This is the scenario most people get wrong intuitively, so it's worth spelling out in full — there are three distinct cases:

**Case 1 — Same filename, same content, same metadata (the normal steady state)**
This is what you get right after Task A's first sync: both sides hold an identical copy. When Task B runs, it compares metadata (size, timestamp) between S3 and NFS — they match, so DataSync marks the file **unchanged** and **skips it**. No transfer, no chatter. Once both sides are in sync, both tasks mostly do nothing on each run except confirm nothing changed.

**Case 2 — Same filename, same content, but metadata differs (e.g. timestamp shifted slightly)**
Can happen from small timing artifacts during a write. DataSync sees the metadata mismatch and **re-transfers the file anyway**, even though the bytes are identical. Harmless, but this is the "sync chatter" mentioned in the trade-offs table above — some wasted transfer and log noise even though nothing meaningful changed.

**Case 3 — Same filename, but genuinely *different* content on each side (the real danger case)**
Say `notes.txt` exists on both NFS and S3, but someone independently edited the NFS copy while someone else independently edited the S3 copy. There's **no merge and no conflict warning** — whichever task's next scheduled run happens first simply overwrites the other side. If Task A (NFS→S3) runs first, it silently overwrites S3's version with the NFS version, discarding whatever was on S3. Task B, running later, now sees NFS and S3 matching (Task A just made them match) and does nothing — so the S3-side edit is gone permanently, with zero indication anything was lost.

**Bottom line:** identical files across both sides are fine and don't cause loops or data loss. The actual risk is only when the *same file* is independently modified on both sides between sync runs — that's a silent last-write-wins overwrite, not a merge, and there's no way to recover the losing version afterward unless S3 versioning is enabled separately on the bucket (which is not part of this lab's setup).

### Setup steps

**Prerequisite:** you already have the NFS Location and S3 Location objects from Phase 10 — you'll reuse both, just swapped.

1. **DataSync console → Tasks → Create task**

2. **Step 1 — Source location:**
   - Choose **an existing location**
   - Select your existing **S3 location** (`datasync-demo-bucket-<yourname>/output/`)
   - Click **Next**

3. **Step 2 — Destination location:**
   - Choose **an existing location**
   - Select your existing **NFS location** (`/data/shared` via your agent)
   - Click **Next**
   - *(Same agent, same NFS server — it's now being written to instead of read from. No new agent needed, but the agent must have network access to write, same as it already does to read.)*

4. **Step 3 — Settings:**
   - **Task mode:** Basic (same constraint as before — must match your agent's mode)
   - **Transfer mode:** Transfer only data that has changed
   - **Verification:** Verify only transferred data
   - **Keep deleted files:** Yes (recommended — see deletion asymmetry note above)
   - **Overwrite files:** Yes
   - **Schedule:** this is the important part — **offset it from Task A's schedule** so the two tasks aren't racing each other every single run. Examples:
     - If Task A runs `cron(0/15 * * * ? *)` (on the hour, :15, :30, :45)
     - Set Task B to `cron(7/15 * * * ? *)` (:07, :22, :37, :52) — roughly halfway between Task A's runs
   - **Logging:** same choice as Task A — bump to detailed if you want to see per-file activity for this direction too

5. **Name it** something distinct, e.g. `datasync-lab-task-reverse`, so the two tasks are easy to tell apart in the console

6. **Step 4 — Review → Create task**

✅ **Checkpoint:** You now have two tasks in the DataSync console:
- `datasync-lab-task` — NFS → S3
- `datasync-lab-task-reverse` — S3 → NFS

Together, a file added on either side eventually appears on both — that's your two-way sync.

### Testing it

```bash
# Test A: change originates on NFS (already covered in Phase 13)
echo "From NFS" > /data/shared/from-nfs.txt
# Wait for Task A's next run → confirm from-nfs.txt appears in S3 /output/

# Test B: change originates on S3
# Upload a file directly to S3 (console, or CLI):
aws s3 cp ./from-s3.txt s3://datasync-demo-bucket-<yourname>/output/from-s3.txt
# Wait for Task B's next run → SSH into the NFS EC2 and confirm it landed in /data/shared/
```

