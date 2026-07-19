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
- Your own public IP address (for the RDP security group rule) — check at [whatismyip.com](https://whatismyip.com).
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
4. AMI: **Windows Server 2022 Base**.
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
3. Same AMI (Windows Server 2022 Base), same instance type, same IAM role (`EC2-S3-Sync-Role` — IAM roles apply account-wide, not per region).
4. Security group in this region: allow RDP (3389) from **My IP** only.
5. Launch and wait for 2/2 status checks.

---

## Step 5 — RDP into the Mumbai instance and create the test file

1. Select `mumbai-sync-test` → **Connect** → **RDP client** tab.
2. **Get password** → upload your `.pem` key → **Decrypt password**.
3. Download the RDP file, connect using an RDP client, log in as `Administrator` with the decrypted password.
4. Open **PowerShell as Administrator** and run:

```powershell
New-Item -Path "C:\TEST" -ItemType Directory
"Hello from Mumbai instance" | Out-File -FilePath "C:\TEST\file.txt"
```

5. Check whether AWS CLI is already present:

```powershell
Test-Path "C:\Program Files\Amazon\AWSCLIV2\aws.exe"
```

- If `True`, try `aws --version`. If that works, skip to step 6 below.
- If `False`, install it:

```powershell
Invoke-WebRequest -Uri "https://awscli.amazonaws.com/AWSCLIV2.msi" -OutFile "$env:TEMP\AWSCLIV2.msi"
Start-Process msiexec.exe -ArgumentList "/i `"$env:TEMP\AWSCLIV2.msi`" /qn" -Wait
```

Confirm the install actually landed on disk:

```powershell
Test-Path "C:\Program Files\Amazon\AWSCLIV2\aws.exe"
```

This should now return `True`. Then check the command:

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

6. Confirm the IAM role is attached and working:

```powershell
aws sts get-caller-identity
aws s3 ls
```

`aws sts get-caller-identity` should return an `assumed-role/EC2-S3-Sync-Role/...` ARN — this confirms the instance is using the IAM role, with no access keys involved. If `aws s3 ls` returns without an access-denied error (an empty result is fine if the bucket has no other content), the IAM role is attached correctly. If you get an error, go back to the instance in the EC2 console → **Actions → Security → Modify IAM role** and attach `EC2-S3-Sync-Role`.

---

## Step 6 — Set up two-way sync (Mumbai)

Still in the Mumbai RDP session, PowerShell as Administrator. Create the scripts folder and both scripts — one pushes local changes up, one pulls remote changes down:

```powershell
New-Item -Path "C:\Scripts" -ItemType Directory -Force

@'
aws s3 sync C:\TEST s3://your-unique-sync-bucket-name/TEST
'@ | Out-File -FilePath "C:\Scripts\sync-up.ps1" -Encoding ascii

@'
aws s3 sync s3://your-unique-sync-bucket-name/TEST C:\TEST
'@ | Out-File -FilePath "C:\Scripts\sync-down.ps1" -Encoding ascii
```

> Replace the bucket name with the one from Step 1, in **both** scripts.

Test the push manually first, before scheduling anything:

```powershell
powershell -ExecutionPolicy Bypass -File C:\Scripts\sync-up.ps1
aws s3 ls s3://your-unique-sync-bucket-name/TEST/
```

You should see `file.txt` listed. Now register **both** scheduled tasks — push runs first, pull runs a few seconds later on the same 2-minute cycle, so this instance's own new files go up before it checks for anything new from the other side:

```powershell
$actionUp = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-ExecutionPolicy Bypass -File C:\Scripts\sync-up.ps1"
$triggerUp = New-ScheduledTaskTrigger -Once -At (Get-Date) -RepetitionInterval (New-TimeSpan -Minutes 2) -RepetitionDuration ([TimeSpan]::MaxValue)
Register-ScheduledTask -TaskName "S3SyncUp" -Action $actionUp -Trigger $triggerUp -User "SYSTEM" -RunLevel Highest

$actionDown = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-ExecutionPolicy Bypass -File C:\Scripts\sync-down.ps1"
$triggerDown = New-ScheduledTaskTrigger -Once -At ((Get-Date).AddSeconds(30)) -RepetitionInterval (New-TimeSpan -Minutes 2) -RepetitionDuration ([TimeSpan]::MaxValue)
Register-ScheduledTask -TaskName "S3SyncDown" -Action $actionDown -Trigger $triggerDown -User "SYSTEM" -RunLevel Highest
```

Verify both tasks are registered:

```powershell
Get-ScheduledTask -TaskName "S3SyncUp","S3SyncDown"
```

---

## Step 7 — Set up two-way sync (Hyderabad)

RDP into `hyderabad-sync-test` the same way as Step 5. Then, PowerShell as Administrator:

```powershell
New-Item -Path "C:\TEST" -ItemType Directory
```

Check AWS CLI here too (each instance is separate — installing it on Mumbai doesn't install it on Hyderabad):

```powershell
Test-Path "C:\Program Files\Amazon\AWSCLIV2\aws.exe"
```

If `False`, install it the same way as Step 5:

```powershell
Invoke-WebRequest -Uri "https://awscli.amazonaws.com/AWSCLIV2.msi" -OutFile "$env:TEMP\AWSCLIV2.msi"
Start-Process msiexec.exe -ArgumentList "/i `"$env:TEMP\AWSCLIV2.msi`" /qn" -Wait
Test-Path "C:\Program Files\Amazon\AWSCLIV2\aws.exe"
aws --version
```

If `Test-Path` is `True` but `aws --version` isn't recognized, fix the current session's PATH (see Step 5 for the full explanation of why this happens):

```powershell
$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
aws --version
```

Confirm the role here too:

```powershell
aws sts get-caller-identity
aws s3 ls
```

Now set up both scripts here too — same pattern as Mumbai:

```powershell
New-Item -Path "C:\Scripts" -ItemType Directory -Force

@'
aws s3 sync C:\TEST s3://your-unique-sync-bucket-name/TEST
'@ | Out-File -FilePath "C:\Scripts\sync-up.ps1" -Encoding ascii

@'
aws s3 sync s3://your-unique-sync-bucket-name/TEST C:\TEST
'@ | Out-File -FilePath "C:\Scripts\sync-down.ps1" -Encoding ascii
```

Test the pull manually first:

```powershell
powershell -ExecutionPolicy Bypass -File C:\Scripts\sync-down.ps1
Get-Content C:\TEST\file.txt
```

You should see `Hello from Mumbai instance` printed. Once confirmed, register **both** scheduled tasks:

```powershell
$actionUp = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-ExecutionPolicy Bypass -File C:\Scripts\sync-up.ps1"
$triggerUp = New-ScheduledTaskTrigger -Once -At (Get-Date) -RepetitionInterval (New-TimeSpan -Minutes 2) -RepetitionDuration ([TimeSpan]::MaxValue)
Register-ScheduledTask -TaskName "S3SyncUp" -Action $actionUp -Trigger $triggerUp -User "SYSTEM" -RunLevel Highest

$actionDown = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-ExecutionPolicy Bypass -File C:\Scripts\sync-down.ps1"
$triggerDown = New-ScheduledTaskTrigger -Once -At ((Get-Date).AddSeconds(30)) -RepetitionInterval (New-TimeSpan -Minutes 2) -RepetitionDuration ([TimeSpan]::MaxValue)
Register-ScheduledTask -TaskName "S3SyncDown" -Action $actionDown -Trigger $triggerDown -User "SYSTEM" -RunLevel Highest
```

Verify both tasks are registered:

```powershell
Get-ScheduledTask -TaskName "S3SyncUp","S3SyncDown"
```

---

## Step 8 — Verify sync works in both directions

**Mumbai → Hyderabad:**

1. On the **Mumbai** RDP session:

```powershell
"Updated from Mumbai - testing sync" | Out-File -FilePath "C:\TEST\file.txt"
```

2. Wait up to 4 minutes (one push cycle + one pull cycle on the Hyderabad side).
3. On the **Hyderabad** RDP session, check:

```powershell
Get-Content C:\TEST\file.txt
```

It should show the updated text without you doing anything manually on Hyderabad.

**Hyderabad → Mumbai (the new direction):**

4. On the **Hyderabad** RDP session, create a brand-new file:

```powershell
"Created on Hyderabad instance" | Out-File -FilePath "C:\TEST\from-hyderabad.txt"
```

5. Wait up to 4 minutes.
6. On the **Mumbai** RDP session, check:

```powershell
Get-Content C:\TEST\from-hyderabad.txt
Get-ChildItem C:\TEST
```

`from-hyderabad.txt` should now exist on Mumbai too, with no manual action taken there. This confirms sync works in both directions — whichever instance creates or edits a file in `C:\TEST`, it shows up on the other side automatically.

---

## Common issues (checked before publishing this guide)

| Symptom | Cause | Fix |
|---|---|---|
| Hyderabad region missing from region dropdown | It's opt-in, not enabled by default | Do Step 0 first |
| `Test-Path "C:\Program Files\Amazon\AWSCLIV2\aws.exe"` returns `True` but `aws --version` says "not recognized" | Install succeeded, but the current PowerShell session already cached the old PATH before install ran | Run the `$env:Path = ...` refresh command shown in Step 5/7 (fixes current session); if it happens in *every new* session too, run the permanent PATH fix also shown in Step 5 |
| `aws s3 ls` → Access Denied | IAM role not attached, or attached after launch without reboot | Re-check instance → Actions → Security → Modify IAM role; if just attached, run `aws s3 ls` again (no reboot needed, role applies live) |
| Scheduled task doesn't run | Task registered under wrong user, or `-ExecutionPolicy Bypass` missing | Re-run the `Register-ScheduledTask` command exactly as shown |
| `RequestTimeTooSkewed` error | Windows clock not synced yet (common right after launch) | Wait a few minutes for Windows Time service to sync, or run `w32tm /resync` |
| File shows old content, or a new file hasn't appeared on the other instance yet | Sync hasn't cycled yet on one or both sides | Wait for the next 2-minute interval, or manually run `sync-up.ps1` on the source instance then `sync-down.ps1` on the other |
| Same filename edited on both instances around the same time, one edit "disappears" | Expected behavior — last sync wins, no conflict resolution (see Notes for learning) | Not a bug. Use different filenames per instance if this matters for your test, or avoid editing the same file from both sides at once |
| Can't RDP in | Security group only allows your IP, and your IP changed | Update the inbound rule to your current IP |

---

## Cost (approximate, ap-south-1 / ap-south-2, on-demand)

| Item | Cost |
|---|---|
| 2× `t3.micro` Windows instances (730 hrs/month each) | ~$15–20/month **each** (Windows licensing is the main driver) |
| 2× 30 GB gp3 EBS root volumes | ~$2.40/month each |
| S3 storage (a text file) | Negligible |
| S3 PUT/LIST requests (every 2 min, 2 tasks × 2 instances) | Still a few cents/month — request pricing is fractions of a cent per 1,000 calls |
| Cross-region data transfer (S3 → Hyderabad EC2) | ~$0.09/GB — negligible for a text file, scales up for real workloads |
| **Total for this lab left running 24/7** | **~$35–45/month** |

**To keep this cheap:** stop (don't just leave running) both instances when you're not actively testing. Stopped instances don't charge for compute, only for the EBS volume (~$2–5/month combined).

---

## Cleanup (do this after you're done — in order)

1. On both instances, unregister both scheduled tasks (optional, instance is being deleted anyway):
   ```powershell
   Unregister-ScheduledTask -TaskName "S3SyncUp" -Confirm:$false
   Unregister-ScheduledTask -TaskName "S3SyncDown" -Confirm:$false
   ```
   (Run this on Mumbai, then run the same two lines again on Hyderabad — both instances now have both tasks.)
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

## Notes for learning

- This is now a **two-way mirror**. If both sides edit the *same filename* near-simultaneously, last sync wins — no merge or conflict resolution. That's expected behavior for this simple approach, not a bug. If you need real conflict handling, that's a meaningfully bigger build (versioned objects + a resolution script, or a proper distributed file system).
- Sync is polling-based (every 2 min per direction), not truly real-time. Real-time would need S3 Event Notifications + Lambda, or AWS DataSync — out of scope for "simplest approach."
- No VPC peering, Transit Gateway, or VPN was needed anywhere in this lab — that's the entire point of routing through S3 instead of a direct network path between the two regions.
- No infinite sync loop risk: `aws s3 sync` only copies files that are new or have a different size/timestamp than what's already there. Once both sides and S3 match, a sync cycle does nothing until a real change happens again.

---

## Appendix — Alternative: IAM User with access keys instead of IAM role

The main lab above uses an IAM role, which is the AWS-recommended method for EC2 (no long-lived keys to manage or leak). If your course/manager specifically wants you to demonstrate `aws configure` with an access key instead, here's the substitution — everything else in the lab (bucket, folders, scripts, scheduled tasks) stays identical.

**Skip Step 2 (IAM role) and instead:**

1. IAM Console → **Users** → **Create user** → name it `EC2-S3-Sync-User`.
2. Do **not** enable console access — this user is for programmatic (CLI) access only.
3. Attach policy: **AmazonS3FullAccess** (or the scoped policy from "Tightening security" above, adjusted to an IAM user instead of a role).
4. After the user is created → **Security credentials** tab → **Create access key** → choose **Command Line Interface (CLI)** as the use case → download the CSV. This contains your `Access Key ID` and `Secret Access Key` — keep it private, don't commit it anywhere.

**In Steps 3 and 4 (launching instances):** skip attaching an IAM instance profile entirely — this method doesn't need one.

**In Step 5 and Step 7, after confirming `aws --version` works, run this instead of checking a role:**

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

Everything from Step 6 onward (folders, both sync scripts, both scheduled tasks, testing in both directions, cleanup) is identical regardless of which authentication method you used.

**Which to use:** IAM role for anything beyond a one-off demo — no keys to rotate or accidentally leak. IAM user + access keys only if you're specifically asked to show that authentication flow, or want to see how `aws configure` and credential files work under the hood.
