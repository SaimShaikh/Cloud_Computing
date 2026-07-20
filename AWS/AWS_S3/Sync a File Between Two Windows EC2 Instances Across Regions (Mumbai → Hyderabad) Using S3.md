# Lab: Sync a File Between Two Windows EC2 Instances Across Regions (Mumbai → Hyderabad) Using S3


<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/6c33e69b-774f-497e-b6bb-26ee5a6026b2" />




## Objective

1. Launch a Windows EC2 instance in **Mumbai (ap-south-1)**.
2. Launch a second Windows EC2 instance in **Hyderabad (ap-south-2)**.
3. On both instances, maintain a `C:\TEST` folder such that **any new or changed file created on either instance automatically appears on the other** — without VPC peering, VPN, or Transit Gateway.

## Approach

Use **S3 as the sync relay**, since it's reachable over the public AWS endpoint from any region with zero networking setup. Both instances run **both directions** of sync on a schedule:

```
                 push (local → S3)                    push (local → S3)
Mumbai EC2  ───────────────────────►   S3 Bucket   ◄───────────────────────  Hyderabad EC2
 C:\TEST    ◄───────────────────────   (TEST/)     ───────────────────────►   C:\TEST
                 pull (S3 → local)                    pull (S3 → local)
```

- Every 2 minutes, each instance pushes its own `C:\TEST` up to S3, then pulls whatever is in S3 down into `C:\TEST`.
- Because push runs right before pull on each cycle, a new file created on Mumbai gets pushed to S3, then picked up by Hyderabad's next pull — and the same in reverse.
- `aws s3 sync` only transfers new/changed files — cheap and simple.
- This is **two-way**. No cross-region networking is required at all.

**Important limitation to know going in:** if both instances create or edit the *same filename* within the same ~2–4 minute window, the last one to sync wins — there's no merge or conflict resolution. This is fine for a lab and for most real "keep folders in sync" use cases, but it's not a true multi-master file system. See **Notes for learning** at the end.

---

## Prerequisites

- An AWS account with permission to create EC2 instances, IAM roles, and S3 buckets.
- A key pair for RDP access (create one during launch if you don't have one).
- Your own public IP address (for the RDP security group rule) — check at whatismyip.com.
- Both instances must be launched in the **default VPC** (or any VPC/subnet with "auto-assign public IP" enabled and a route to an Internet Gateway). This lab reaches S3 over its public endpoint, so the instance needs outbound internet access. If you use the default VPC without changing anything, this is already true — just don't switch off "Auto-assign public IP" in the launch wizard.

---

## Step 0 — Enable the Hyderabad region

Mumbai (`ap-south-1`) is enabled by default. Hyderabad (`ap-south-2`) is an **opt-in region** and will not appear until you enable it.

1. Click your account name (top right) → **Account**.
2. Scroll to **AWS Regions**.
3. Find **Asia Pacific (Hyderabad) ap-south-2** → click **Enable**.
4. Confirm. It takes a few minutes to activate.

---

## Step 1 — Create the S3 bucket

Bucket names are global, so this only needs to be done once. Since the instances don't exist yet, run this from your **local Windows PC** (in PowerShell, with AWS CLI installed and `aws configure` already run using your own AWS account credentials):

```powershell
aws s3 mb s3://your-unique-sync-bucket-name --region ap-south-1
```

> Replace `your-unique-sync-bucket-name` with something globally unique (e.g. `saime-ec2-sync-lab-2026`). Use this exact name in every step below.

**No AWS CLI on your local PC?** Skip the command above and use the console instead: S3 Console → **Create bucket** → Region: **Asia Pacific (Mumbai) ap-south-1** → enter the bucket name → leave every other setting at default → **Create bucket**.

No further bucket configuration is needed. Default settings (Block Public Access on, ACLs disabled, default encryption) don't interfere with this lab — the instances authenticate via IAM role, not public access, so `aws s3 sync` works against the bucket exactly as created.

**Optional — safety net:** enable versioning so an overwritten file can be recovered later (not required for the lab to work):

```powershell
aws s3api put-bucket-versioning --bucket your-unique-sync-bucket-name --versioning-configuration Status=Enabled
```

(Or via console: open the bucket → **Properties** tab → **Bucket Versioning** → **Enable**.)

---

## Step 2 — Create an IAM role for the EC2 instances

Both instances need permission to read/write this one bucket. IAM is global, so one role works in both regions.

1. IAM Console → **Roles** → **Create role**.
2. Trusted entity type: **AWS service** → Use case: **EC2**.
3. Attach policy: for this lab, use **AmazonS3FullAccess** (simplest for hands-on; see "Tightening security" below for the production-safe version).
4. Name it: `EC2-S3-Sync-Role`.
5. Create role.

---

## Step 3 — Launch the Mumbai instance

1. EC2 Console → confirm region is **Asia Pacific (Mumbai) ap-south-1** (top right).
2. **Launch instance**.
3. Name: `mumbai-sync-test`.
4. AMI: **Windows Server 2022 Base** (or 2025 Base — both work; see the CLI-installer note in Step 5 if using 2025).
5. Instance type: `t3.micro`.
6. Key pair: select/create one.
7. Network settings → Security group: allow **RDP (3389)** from **My IP** only.
8. Advanced details → **IAM instance profile** → select `EC2-S3-Sync-Role`.
9. Launch instance.
10. Wait for **Status check: 2/2 passed** before continuing (Instance state = running is not enough).

---

## Step 4 — Launch the Hyderabad instance

Repeat Step 3, but:

1. Switch region (top right) to **Asia Pacific (Hyderabad) ap-south-2** first.
2. Name: `hyderabad-sync-test`.
3. Same AMI, same instance type, same IAM role (`EC2-S3-Sync-Role` — IAM roles apply account-wide, not per region).
4. Security group in this region: allow RDP (3389) from **My IP** only.
5. Launch and wait for 2/2 status checks.

---

## Step 5 — RDP into the Mumbai instance, install AWS CLI, verify access, then create the test folder/file

**Order matters here — set up and verify the tooling first, before you create anything locally.** There's no point creating test files and folders if the CLI isn't installed or the IAM role isn't actually working; better to find that out immediately.

1. Select `mumbai-sync-test` → **Connect** → **RDP client** tab.
2. **Get password** → upload your `.pem` key → **Decrypt password**.
3. Download the RDP file, connect using an RDP client, log in as `Administrator` with the decrypted password.
4. Open **PowerShell as Administrator**.

### 5a. Install / verify AWS CLI

Check whether AWS CLI is already present (recent Windows AMIs, especially 2025 Server, ship with it preinstalled):

```powershell
aws --version
```

- If this prints a version (e.g. `aws-cli/2.36.2 ...`), skip straight to **5b**.
- If it says "not recognized," install it:

```powershell
Invoke-WebRequest -Uri "https://awscli.amazonaws.com/AWSCLIV2.msi" -OutFile "$env:TEMP\AWSCLIV2.msi"
Start-Process msiexec.exe -ArgumentList "/i `"$env:TEMP\AWSCLIV2.msi`" /qn" -Wait
```

Confirm the install actually landed on disk:

```powershell
Test-Path "C:\Program Files\Amazon\AWSCLIV2\aws.exe"
```

This should now return `True`. Then check the command again:

```powershell
aws --version
```

**If `Test-Path` is `True` but `aws --version` says "not recognized"** — this is the single most common snag, and it's not a broken install. The installer updated the machine's PATH variable, but your *current* PowerShell session already loaded its PATH into memory before the install ran, so it can't see the change yet. Fix the current session without closing it:

```powershell
$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
aws --version
```

This should now work. To confirm the fix is permanent (so every *future* PowerShell session — after a reboot, a new RDP login, etc. — also finds `aws` without needing that fix again), check that the AWS CLI directory is actually in the **machine-level** PATH, not just your current session:

```powershell
[System.Environment]::GetEnvironmentVariable("Path","Machine") -like "*AWSCLIV2*"
```

If this returns `False` (rare, but possible if the installer didn't fully register PATH), add it permanently yourself:

```powershell
$machinePath = [System.Environment]::GetEnvironmentVariable("Path","Machine")
if ($machinePath -notlike "*C:\Program Files\Amazon\AWSCLIV2*") {
    [System.Environment]::SetEnvironmentVariable("Path", $machinePath + ";C:\Program Files\Amazon\AWSCLIV2", "Machine")
}
```

Open a brand-new PowerShell window and confirm `aws --version` works with no extra steps — that's the real test of whether it's permanent.

### 5b. Verify the IAM role is attached and working

```powershell
aws sts get-caller-identity
aws s3 ls
```

`aws sts get-caller-identity` should return an `assumed-role/EC2-S3-Sync-Role/...` ARN — this confirms the instance is using the IAM role, with no access keys involved. If `aws s3 ls` returns without an access-denied error (an empty result is fine if the bucket has no other content), the IAM role is attached correctly. If you get an error, go back to the instance in the EC2 console → **Actions → Security → Modify IAM role** and attach `EC2-S3-Sync-Role`.

**Do not move on until both commands above succeed.** If the CLI or the role isn't working yet, everything after this point will fail for reasons that look unrelated (e.g. "Access Denied" on the sync script), so it's worth confirming now.

### 5c. Now create the folders and the test file

Only once 5a and 5b both check out:

```powershell
New-Item -Path "C:\TEST" -ItemType Directory
New-Item -Path "C:\Scripts" -ItemType Directory
"Hello from Mumbai instance" | Out-File -FilePath "C:\TEST\file.txt"
```

---

## Step 6 — Create the sync script and test it manually (Mumbai)

Still in the Mumbai RDP session, PowerShell as Administrator:

```powershell
@'
aws s3 sync C:\TEST s3://your-unique-sync-bucket-name/TEST
aws s3 sync s3://your-unique-sync-bucket-name/TEST C:\TEST
'@ | Set-Content C:\Scripts\sync.ps1
```

> Replace the bucket name with the one from Step 1.

**What this script does, line by line:**
- Line 1 (`C:\TEST → s3://.../TEST`) — pushes anything new or changed in the local folder up to S3.
- Line 2 (`s3://.../TEST → C:\TEST`) — pulls anything new or changed in S3 down to the local folder.
- Both lines run every time the task fires, in this order, so a file you just created locally goes up before the pull step checks for anything from the other instance.

Verify the file was created correctly:

```powershell
Test-Path C:\Scripts\sync.ps1
Get-Content C:\Scripts\sync.ps1
```

Test it once, manually, before scheduling anything:

```powershell
powershell.exe -ExecutionPolicy Bypass -File C:\Scripts\sync.ps1
aws s3 ls s3://your-unique-sync-bucket-name/TEST/
```

You should see `file.txt` listed with no errors.

---

## Step 7 — Create the scheduled task (Mumbai)

> **PowerShell vs CMD — read this before you type anything.** The line-continuation character is different in each shell: CMD uses `^`, PowerShell uses `` ` `` (backtick). If you paste a command written for CMD into a PowerShell prompt, PowerShell doesn't recognize `^` as a continuation — it runs the first line, then tries to run `^` and every following line as *separate commands*, which produces a wall of `CommandNotFoundException` errors on `/TN`, `/SC`, `/MO`, etc. This is the single most common error in this step. Two ways to avoid it entirely:

**Option A — one single line (safest, works in both shells):**

```powershell
schtasks /Create /TN "S3FolderSync" /SC MINUTE /MO 2 /RU SYSTEM /RL HIGHEST /TR "powershell.exe -ExecutionPolicy Bypass -File C:\Scripts\sync.ps1" /F
```

**Option B — multi-line, PowerShell-only, using the backtick:**

```powershell
schtasks /Create `
 /TN "S3FolderSync" `
 /SC MINUTE `
 /MO 2 `
 /RU SYSTEM `
 /RL HIGHEST `
 /TR "powershell.exe -ExecutionPolicy Bypass -File C:\Scripts\sync.ps1" `
 /F
```

This creates a task that:
- ✅ Runs every 2 minutes
- ✅ Runs as SYSTEM
- ✅ Runs with highest privileges
- ✅ Starts automatically after reboots (it's a registered scheduled task, not a session-bound job)
- ✅ Needs no Task Scheduler GUI at all

Verify it was created:

```powershell
schtasks /Query /TN "S3FolderSync"
```

or

```powershell
Get-ScheduledTask -TaskName S3FolderSync
```

Run it immediately to confirm it works, rather than waiting 2 minutes:

```powershell
schtasks /Run /TN "S3FolderSync"
```

Check the last run result:

```powershell
Get-ScheduledTaskInfo -TaskName S3FolderSync
```

You should see `LastTaskResult : 0` — `0` means the task completed successfully. Anything else means the task ran but failed; re-check the script path and bucket name in `C:\Scripts\sync.ps1`.

---

## Step 8 — RDP into Hyderabad and repeat Steps 5–7 (same order: CLI and role first, folders after)

RDP into `hyderabad-sync-test` the same way as Step 5. Same order applies here too — verify the tooling and access before creating anything locally.

### 8a. Install / verify AWS CLI

Each instance is separate — installing AWS CLI on Mumbai does **not** install it on Hyderabad, so this has to be checked again from scratch:

```powershell
aws --version
```

Install if needed (same commands, and same PATH troubleshooting, as Step 5a).

### 8b. Verify the IAM role is attached and working

```powershell
aws sts get-caller-identity
aws s3 ls
```

Same as Step 5b — don't move on until both of these succeed.

### 8c. Now create the folders

Only once 8a and 8b both check out:

```powershell
New-Item -Path "C:\TEST" -ItemType Directory -Force
New-Item -Path "C:\Scripts" -ItemType Directory -Force
```

### 8d. Create the sync script

**Do not skip creating the script on this instance.** A very common mistake at this point: the scheduled task command is copy-pasted and run successfully, but the referenced script was never created here — only on Mumbai. This produces exactly this error when you test it:

```
The argument 'C:\Scripts\sync.ps1' to the -File parameter does not exist. Provide the path to an existing '.ps1' file as an argument to the -File parameter.
```

The fix is simply to actually create the file on **this** instance too:

```powershell
@'
aws s3 sync C:\TEST s3://your-unique-sync-bucket-name/TEST
aws s3 sync s3://your-unique-sync-bucket-name/TEST C:\TEST
'@ | Set-Content C:\Scripts\sync.ps1

Test-Path C:\Scripts\sync.ps1
Get-Content C:\Scripts\sync.ps1
```

`Test-Path` must return `True` before you go any further. Then test the pull manually:

```powershell
powershell.exe -ExecutionPolicy Bypass -File C:\Scripts\sync.ps1
Get-ChildItem C:\TEST
Get-Content C:\TEST\file.txt
```

Expected: `file.txt` exists and contains `Hello from Mumbai instance`.

Now create the scheduled task on Hyderabad — same one-line command as Step 7, Option A (safest, avoids the `^` vs backtick trap entirely):

```powershell
schtasks /Create /TN "S3FolderSync" /SC MINUTE /MO 2 /RU SYSTEM /RL HIGHEST /TR "powershell.exe -ExecutionPolicy Bypass -File C:\Scripts\sync.ps1" /F
```

Verify:

```powershell
schtasks /Query /TN "S3FolderSync"
Get-ScheduledTaskInfo -TaskName S3FolderSync
```

---

## Step 9 — Verify sync works in both directions

**Mumbai → Hyderabad:**

1. On the **Mumbai** RDP session:

```powershell
"Test from Mumbai $(Get-Date)" | Out-File -FilePath "C:\TEST\mumbai-test.txt"
```

2. Wait 2–3 minutes (the task runs every 2 minutes on each side).
3. On the **Hyderabad** RDP session:

```powershell
Get-ChildItem C:\TEST
Get-Content C:\TEST\mumbai-test.txt
```

`mumbai-test.txt` should now exist on Hyderabad with no manual action taken there.

**Hyderabad → Mumbai:**

4. On the **Hyderabad** RDP session:

```powershell
"Test from Hyderabad $(Get-Date)" | Out-File -FilePath "C:\TEST\hyderabad-test.txt"
```

5. Wait 2–3 minutes.
6. On the **Mumbai** RDP session:

```powershell
Get-ChildItem C:\TEST
Get-Content C:\TEST\hyderabad-test.txt
```

`hyderabad-test.txt` should now exist on Mumbai too.

**If a file doesn't appear after waiting:** run the script manually on the source instance first (`powershell.exe -ExecutionPolicy Bypass -File C:\Scripts\sync.ps1`) to rule out a scheduling issue vs. a script/permissions issue. If manual sync works but the scheduled task doesn't, re-check `Get-ScheduledTaskInfo -TaskName S3FolderSync` for a non-zero `LastTaskResult`.

If both tests pass, your automated, two-way, cross-region file synchronization is fully working.

---

## Folder structure (end state, both instances)

```
C:\
├── TEST\
│   ├── file.txt
│   ├── mumbai-test.txt          (created on Mumbai, synced to Hyderabad)
│   └── hyderabad-test.txt       (created on Hyderabad, synced to Mumbai)
└── Scripts\
    └── sync.ps1                 (identical content on both instances)
```

S3 bucket layout:

```
s3://your-unique-sync-bucket-name/
└── TEST/
    ├── file.txt
    ├── mumbai-test.txt
    └── hyderabad-test.txt
```

---

## Common issues (every error actually encountered while building this lab)

| Symptom | Cause | Fix |
|---|---|---|
| Hyderabad region missing from region dropdown | It's opt-in, not enabled by default | Do Step 0 first |
| `aws --version` says "not recognized" right after install, even though the MSI ran with no error | Installer updated machine PATH, but the current PowerShell session already cached the old PATH in memory | Run `$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")` then retry `aws --version` |
| `schtasks /Create ^ ... ERROR: Invalid argument/option - '^'` followed by every subsequent line failing as `CommandNotFoundException` on `/TN`, `/SC`, `/MO`, `/RU`, `/RL`, `/TR` | A CMD-style command (using `^` for line continuation) was pasted into a **PowerShell** prompt. PowerShell doesn't understand `^`, so it treats each line as a brand-new, separate command | Either put the whole command on **one line**, or use the PowerShell continuation character `` ` `` (backtick) instead of `^` |
| `The argument 'C:\Scripts\sync.ps1' to the -File parameter does not exist` when testing the scheduled task | The `sync.ps1` script was created on one instance but the scheduled task test is being run on the **other** instance, where the script was never created | Create `C:\Scripts\sync.ps1` on **that specific instance** (each instance needs its own copy — creating it on Mumbai does not create it on Hyderabad) |
| `aws s3 ls` → Access Denied | IAM role not attached, or attached after launch | Re-check instance → Actions → Security → Modify IAM role; no reboot needed, role applies live |
| `schtasks /Query` shows the task but `LastTaskResult` is not `0` | Script ran but failed — usually a wrong bucket name or missing folder | Run the script manually first (`powershell.exe -ExecutionPolicy Bypass -File C:\Scripts\sync.ps1`) and read the actual error output |
| `RequestTimeTooSkewed` error | Windows clock not synced yet (common right after launch) | Wait a few minutes for Windows Time service to sync, or run `w32tm /resync` |
| File shows old content, or a new file hasn't appeared on the other instance yet | Sync hasn't cycled yet on one or both sides | Wait for the next 2-minute interval, or run `C:\Scripts\sync.ps1` manually on both sides |
| Same filename edited on both instances around the same time, one edit "disappears" | Expected behavior — last sync wins, no conflict resolution | Not a bug — see Notes for learning |
| Can't RDP in | Security group only allows your IP, and your IP changed | Update the inbound rule to your current IP |

---

## Cost (approximate, ap-south-1 / ap-south-2, on-demand)

| Item | Cost |
|---|---|
| 2× `t3.micro` Windows instances (730 hrs/month each) | ~$15–20/month **each** (Windows licensing is the main driver) |
| 2× 30 GB gp3 EBS root volumes | ~$2.40/month each |
| S3 storage (a few text files) | Negligible |
| S3 PUT/LIST requests (every 2 min × 2 instances) | A few cents/month |
| Cross-region data transfer (S3 → Hyderabad EC2) | ~$0.09/GB — negligible for text files |
| **Total for this lab left running 24/7** | **~$35–45/month** |

**To keep this cheap:** stop (don't just leave running) both instances when you're not actively testing. Stopped instances don't charge for compute, only for the EBS volume (~$2–5/month combined).

---

## Cleanup (do this after you're done — in order)

1. On both instances, remove the scheduled task (optional, instance is being deleted anyway):
   ```powershell
   schtasks /Delete /TN "S3FolderSync" /F
   ```
2. EC2 Console (Mumbai) → terminate `mumbai-sync-test`.
3. EC2 Console (Hyderabad) → terminate `hyderabad-sync-test`.
4. IAM Console → delete role `EC2-S3-Sync-Role`.
5. Empty and delete the S3 bucket (from your local Windows PC, in PowerShell):
   ```powershell
   aws s3 rm s3://your-unique-sync-bucket-name --recursive
   aws s3 rb s3://your-unique-sync-bucket-name
   ```
   Or via console: open the bucket → select all objects → **Delete** → type `permanently delete` to confirm → then go back to the bucket list → select the bucket → **Delete** → type the bucket name to confirm.

---

## Tightening security (optional, do this if you want production-safe habits)

Replace `AmazonS3FullAccess` in Step 2 with a scoped inline policy limited to just this bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::your-unique-sync-bucket-name"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::your-unique-sync-bucket-name/*"
    }
  ]
}
```

---

## Advantages of this approach

- No VPC peering, Transit Gateway, or VPN needed anywhere — S3's public regional endpoint is reachable from any region by default.
- Extremely simple to set up and explain — two `aws s3 sync` commands and one scheduled task per instance.
- `aws s3 sync` is idempotent and cheap: it only transfers files that are new or have a different size/timestamp, so a steady-state sync cycle with no changes does effectively nothing.
- Easy to demo live in an interview — create a file, wait, show it appearing on the other instance.

## Limitations

- **Last-writer-wins, no conflict resolution.** If both sides edit the same filename within the same sync window, whichever instance's push lands last in S3 overwrites the other.
- **Polling-based, not real-time.** With a 2-minute schedule, worst-case propagation delay is just under 4 minutes (source instance's next push + destination instance's next pull).
- **Deletions are not synchronized** by default. `aws s3 sync` does not remove files at the destination unless you add the `--delete` flag to both lines in `sync.ps1`. If exact mirroring (including deletes) is required, add `--delete` to each `aws s3 sync` line — but test this carefully, since a delete on one side will now propagate and remove the file everywhere.

## Future improvements

- Add `--delete` to both sync directions if exact mirroring is required, with a clear understanding that deletes now propagate.
- Add logging (`Start-Transcript` or redirect output to `C:\Scripts\sync.log`) so scheduled runs can be audited without needing to run the script manually.
- Move from polling to event-driven sync using **S3 Event Notifications → SQS/Lambda** for near-real-time propagation instead of a fixed 2-minute interval.
- For a true multi-master file system with conflict resolution, look at **AWS DataSync** or a purpose-built distributed file system instead of hand-rolled `aws s3 sync` scripts.

## Notes for learning

- This is a **two-way mirror**. If both sides edit the *same filename* near-simultaneously, last sync wins — no merge or conflict resolution. That's expected behavior for this simple approach, not a bug.
- Sync is polling-based (every 2 minutes), not truly real-time. Real-time would need S3 Event Notifications + Lambda, or AWS DataSync — out of scope for "simplest approach."
- No VPC peering, Transit Gateway, or VPN was needed anywhere in this lab — that's the entire point of routing through S3 instead of a direct network path between the two regions.
- No infinite sync loop risk: `aws s3 sync` only copies files that are new or have a different size/timestamp than what's already there.

---

## Interview Q&A for this project

**Q: Why did you use S3 instead of VPC Peering or a Transit Gateway to connect two regions?**
A: The goal was file-level synchronization, not full network connectivity. S3 already has a public regional endpoint reachable from anywhere with an internet route, so there was no need to build and pay for a private network path just to move files between two folders. VPC Peering/TGW would be the right call if the workloads needed to talk over arbitrary ports/protocols — this only needed object storage as a relay.

**Q: How does data get from Mumbai to Hyderabad without a direct connection between the two instances?**
A: Neither instance ever talks to the other directly. Each instance independently pushes its local folder to a shared S3 bucket and pulls whatever's in that bucket back down, on a schedule. S3 is the single shared point of contact; the two EC2 instances never establish a connection with each other.

**Q: Why IAM role instead of access keys?**
A: An IAM role attached to the instance profile provides temporary, automatically-rotated credentials with no secrets to store, distribute, or leak. Access keys are long-lived and have to be manually rotated and protected — a role is the AWS-recommended pattern for anything running on EC2 that needs to call AWS APIs.

**Q: What happens if both instances edit the same file at the same time?**
A: Whichever instance's push completes last in that sync cycle wins — `aws s3 sync` has no merge logic, so the earlier write is silently overwritten. This is an explicit limitation of the design, not a bug; a production system needing real conflict resolution would need versioned objects and custom merge logic, or a different architecture entirely.

**Q: Why does `aws s3 sync` not create excessive S3 costs even running every 2 minutes?**
A: `aws s3 sync` compares file size and last-modified timestamp before transferring anything — if nothing changed since the last run, it makes only a LIST call and no PUT/GET calls. LIST/PUT/GET requests are priced in fractions of a cent per thousand calls, so even continuous polling of a small folder costs a few cents a month.

**Q: How would you make this real-time instead of every 2 minutes?**
A: Replace the polling scheduled task with an event-driven pipeline — S3 Event Notifications firing on `PutObject`, delivered to SQS or triggering a Lambda function that immediately pushes the change to the other side (or notifies an agent on the instance to pull immediately). This removes the polling delay entirely at the cost of more moving parts.

**Q: Why did the scheduled task fail with errors on `/TN`, `/SC`, `/MO` even though the command worked for someone else?**
A: That command was written using `^` as a line-continuation character, which is CMD syntax. In a PowerShell prompt, `^` isn't a continuation character, so PowerShell executes the first line and then tries to run each subsequent line (`/TN "S3FolderSync"`, etc.) as its own standalone command — which fails because none of those are valid PowerShell commands or cmdlets. The fix is to either put the whole command on one line or use PowerShell's own continuation character, the backtick.

**Q: How would you scale this beyond two regions/instances?**
A: The same push/pull pattern generalizes — every additional instance just needs the same IAM role, the same script, and the same scheduled task pointed at the same bucket/prefix. The main thing to watch as the fleet grows is the last-writer-wins conflict model getting riskier with more simultaneous writers, which is when a move to a real conflict-resolution or locking mechanism becomes worthwhile.

---

## Appendix — Alternative: IAM User with access keys instead of IAM role

The main lab above uses an IAM role, which is the AWS-recommended method for EC2 (no long-lived keys to manage or leak). If your course/manager specifically wants you to demonstrate `aws configure` with an access key instead, here's the substitution — everything else in the lab (bucket, folders, script, scheduled task) stays identical.

**Skip Step 2 (IAM role) and instead:**

1. IAM Console → **Users** → **Create user** → name it `EC2-S3-Sync-User`.
2. Do **not** enable console access — this user is for programmatic (CLI) access only.
3. Attach policy: **AmazonS3FullAccess** (or the scoped policy from "Tightening security" above, adjusted to an IAM user instead of a role).
4. After the user is created → **Security credentials** tab → **Create access key** → choose **Command Line Interface (CLI)** as the use case → download the CSV. This contains your `Access Key ID` and `Secret Access Key` — keep it private, don't commit it anywhere.

**In Steps 3 and 4 (launching instances):** skip attaching an IAM instance profile entirely — this method doesn't need one.

**In Step 5 and Step 8, after confirming `aws --version` works, run this instead of checking a role:**

```powershell
aws configure
```

Enter when prompted:
```
AWS Access Key ID: <your access key>
AWS Secret Access Key: <your secret key>
Default region name: ap-south-1     (use ap-south-2 on the Hyderabad instance)
Default output format: json
```

Then verify:

```powershell
aws sts get-caller-identity
```

Expected output should show `"Arn": ".../user/EC2-S3-Sync-User"` (a **user** ARN, not an **assumed-role** ARN — this is how you can tell the two methods apart when checking your work).

Credentials are stored at `C:\Users\Administrator\.aws\credentials` and `C:\Users\Administrator\.aws\config`. You can inspect them with:

```powershell
notepad $env:USERPROFILE\.aws\credentials
```

Everything from Step 6 onward (script, scheduled task, testing in both directions, cleanup) is identical regardless of which authentication method you used.

**Which to use:** IAM role for anything beyond a one-off demo — no keys to rotate or accidentally leak. IAM user + access keys only if you're specifically asked to show that authentication flow, or want to see how `aws configure` and credential files work under the hood.
