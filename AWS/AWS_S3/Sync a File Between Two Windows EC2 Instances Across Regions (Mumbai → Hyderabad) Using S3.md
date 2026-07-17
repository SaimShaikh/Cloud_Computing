# Lab: Sync a File Between Two Windows EC2 Instances Across Regions (Mumbai → Hyderabad) Using S3


<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/6c33e69b-774f-497e-b6bb-26ee5a6026b2" />

## Objective

1. Launch a Windows EC2 instance in **Mumbai (ap-south-1)**.
2. Create `C:\TEST\file.txt` on it.
3. Launch a second Windows EC2 instance in **Hyderabad (ap-south-2)**.
4. Automatically get that file reflected in `C:\TEST` on the Hyderabad instance — without VPC peering, VPN, or Transit Gateway.

## Approach

Use **S3 as the sync relay**, since it's reachable over the public AWS endpoint from any region with zero networking setup:

```
Mumbai EC2 (C:\TEST) --[Task Scheduler: aws s3 sync UP]--> S3 Bucket --[Task Scheduler: aws s3 sync DOWN]--> Hyderabad EC2 (C:\TEST)
```

- Mumbai instance pushes `C:\TEST` to S3 every 2 minutes.
- Hyderabad instance pulls from S3 into `C:\TEST` every 2 minutes.
- `aws s3 sync` only transfers new/changed files — cheap and simple.
- This is **one-way** (Mumbai → Hyderabad). No cross-region networking is required at all.

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

Bucket names are global, so this only needs to be done once, in either region. Do it from your local machine or CloudShell (Mumbai console):

```bash
aws s3 mb s3://your-unique-sync-bucket-name --region ap-south-1
```

> Replace `your-unique-sync-bucket-name` with something globally unique (e.g. `saime-ec2-sync-lab-2026`). Use this exact name in every step below.

No further bucket configuration is needed. Default settings (Block Public Access on, ACLs disabled, default encryption) don't interfere with this lab — the instances authenticate via IAM role, not public access, so `aws s3 sync` works against the bucket exactly as created.

**Optional — safety net:** enable versioning so an overwritten file can be recovered later (not required for the lab to work):

```bash
aws s3api put-bucket-versioning --bucket your-unique-sync-bucket-name --versioning-configuration Status=Enabled
```

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
aws --version
```

- If it prints a version number, skip to the next check.
- If it says the command is not recognized, install it (Windows Server AMIs include AWS Tools for PowerShell by default, but the standalone `aws` CLI is not guaranteed to be present on every AMI build):

```powershell
Start-Process msiexec.exe -ArgumentList "/i https://awscli.amazonaws.com/AWSCLIV2.msi /qn" -Wait
```

Close and reopen PowerShell as Administrator afterward (so the updated PATH is picked up), then confirm:

```powershell
aws --version
```

6. Confirm the IAM role is attached and working:

```powershell
aws s3 ls
```

If this returns without an access-denied error (an empty result is fine if the bucket has no other content), the IAM role is attached correctly. If you get an error, go back to the instance in the EC2 console → **Actions → Security → Modify IAM role** and attach `EC2-S3-Sync-Role`.

---

## Step 6 — Set up the push sync (Mumbai)

Still in the Mumbai RDP session, PowerShell as Administrator:

```powershell
New-Item -Path "C:\Scripts" -ItemType Directory -Force

@'
aws s3 sync C:\TEST s3://your-unique-sync-bucket-name/TEST
'@ | Out-File -FilePath "C:\Scripts\sync-up.ps1" -Encoding ascii
```

> Replace the bucket name with the one from Step 1.

Test it manually first, before scheduling:

```powershell
powershell -ExecutionPolicy Bypass -File C:\Scripts\sync-up.ps1
aws s3 ls s3://your-unique-sync-bucket-name/TEST/
```

You should see `file.txt` listed. Once confirmed, register the scheduled task:

```powershell
$action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-ExecutionPolicy Bypass -File C:\Scripts\sync-up.ps1"
$trigger = New-ScheduledTaskTrigger -Once -At (Get-Date) -RepetitionInterval (New-TimeSpan -Minutes 2) -RepetitionDuration ([TimeSpan]::MaxValue)
Register-ScheduledTask -TaskName "S3SyncUp" -Action $action -Trigger $trigger -User "SYSTEM" -RunLevel Highest
```

---

## Step 7 — Set up the pull sync (Hyderabad)

RDP into `hyderabad-sync-test` the same way as Step 5. Then, PowerShell as Administrator:

```powershell
New-Item -Path "C:\TEST" -ItemType Directory
New-Item -Path "C:\Scripts" -ItemType Directory -Force

@'
aws s3 sync s3://your-unique-sync-bucket-name/TEST C:\TEST
'@ | Out-File -FilePath "C:\Scripts\sync-down.ps1" -Encoding ascii
```

Test manually first:

```powershell
powershell -ExecutionPolicy Bypass -File C:\Scripts\sync-down.ps1
Get-Content C:\TEST\file.txt
```

You should see `Hello from Mumbai instance` printed. Once confirmed, register the scheduled task:

```powershell
$action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-ExecutionPolicy Bypass -File C:\Scripts\sync-down.ps1"
$trigger = New-ScheduledTaskTrigger -Once -At (Get-Date) -RepetitionInterval (New-TimeSpan -Minutes 2) -RepetitionDuration ([TimeSpan]::MaxValue)
Register-ScheduledTask -TaskName "S3SyncDown" -Action $action -Trigger $trigger -User "SYSTEM" -RunLevel Highest
```

---

## Step 8 — Verify end-to-end automation

1. Go back to the **Mumbai** RDP session. Edit the file:

```powershell
"Updated from Mumbai - testing automatic sync" | Out-File -FilePath "C:\TEST\file.txt"
```

2. Wait up to 4 minutes (one push cycle + one pull cycle).
3. On the **Hyderabad** RDP session, check:

```powershell
Get-Content C:\TEST\file.txt
```

It should show the updated text without you doing anything manually on the Hyderabad side. This confirms automatic cross-region reflection.

---

## Common issues (checked before publishing this guide)

| Symptom | Cause | Fix |
|---|---|---|
| Hyderabad region missing from region dropdown | It's opt-in, not enabled by default | Do Step 0 first |
| `aws s3 ls` → Access Denied | IAM role not attached, or attached after launch without reboot | Re-check instance → Actions → Security → Modify IAM role; if just attached, run `aws s3 ls` again (no reboot needed, role applies live) |
| Scheduled task doesn't run | Task registered under wrong user, or `-ExecutionPolicy Bypass` missing | Re-run the `Register-ScheduledTask` command exactly as shown |
| `RequestTimeTooSkewed` error | Windows clock not synced yet (common right after launch) | Wait a few minutes for Windows Time service to sync, or run `w32tm /resync` |
| File shows old content on Hyderabad | Sync hasn't cycled yet | Wait for the next 2-minute interval, or manually run `sync-down.ps1` |
| Can't RDP in | Security group only allows your IP, and your IP changed | Update the inbound rule to your current IP |

---

## Cost (approximate, ap-south-1 / ap-south-2, on-demand)

| Item | Cost |
|---|---|
| 2× `t3.micro` Windows instances (730 hrs/month each) | ~$15–20/month **each** (Windows licensing is the main driver) |
| 2× 30 GB gp3 EBS root volumes | ~$2.40/month each |
| S3 storage (a text file) | Negligible |
| S3 PUT/LIST requests (every 2 min, both directions) | A few cents/month |
| Cross-region data transfer (S3 → Hyderabad EC2) | ~$0.09/GB — negligible for a text file, scales up for real workloads |
| **Total for this lab left running 24/7** | **~$35–45/month** |

**To keep this cheap:** stop (don't just leave running) both instances when you're not actively testing. Stopped instances don't charge for compute, only for the EBS volume (~$2–5/month combined).

---

## Cleanup (do this after you're done — in order)

1. On both instances, unregister the scheduled tasks (optional, instance is being deleted anyway):
   ```powershell
   Unregister-ScheduledTask -TaskName "S3SyncUp" -Confirm:$false   # Mumbai
   Unregister-ScheduledTask -TaskName "S3SyncDown" -Confirm:$false # Hyderabad
   ```
2. EC2 Console (Mumbai) → terminate `mumbai-sync-test`.
3. EC2 Console (Hyderabad) → terminate `hyderabad-sync-test`.
4. IAM Console → delete role `EC2-S3-Sync-Role`.
5. Empty and delete the S3 bucket:
   ```bash
   aws s3 rm s3://your-unique-sync-bucket-name --recursive
   aws s3 rb s3://your-unique-sync-bucket-name
   ```

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

- This is a **one-way mirror** (Mumbai → Hyderabad). If both sides edit the same file near-simultaneously, last sync wins — no conflict resolution. That's expected for this lab, not a bug.
- Sync is polling-based (every 2 min), not truly real-time. Real-time would need S3 Event Notifications + Lambda, or AWS DataSync — out of scope for "simplest approach."
- No VPC peering, Transit Gateway, or VPN was needed anywhere in this lab — that's the entire point of routing through S3 instead of a direct network path between the two regions.
