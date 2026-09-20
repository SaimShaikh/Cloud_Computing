# AWS Transfer Family — SFTP Server: Definitive Hands-On Lab (Windows + Mac)

> This is the final, debugged version of this lab — every step here has been tested end-to-end, and every real issue hit during testing has a documented fix built directly into the relevant step. Follow it top to bottom and you should not hit any surprises.

---

## 1. Scenario

Build one AWS Transfer Family SFTP server backed by S3, then connect to it and transfer files (text, images, any file type) from **both Windows and Mac**, using either the command line or a GUI client.

---

## 2. Architecture

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/43ee7062-31b6-487b-b0f5-63f7a2676201" />


---

## 3. Client Applications — Which One for Which OS

| OS | Application | Type | Notes |
|---|---|---|---|
| Windows | **MobaXterm** | Free/Paid | Terminal + drag-and-drop SFTP browser in one. Accepts OpenSSH keys directly — no conversion needed. Windows-only. |
| Windows | **WinSCP** | Free | Dedicated GUI SFTP client. Converts OpenSSH keys to `.ppk` on first use — accept the prompt. |
| Windows/Mac | **FileZilla** | Free | Cross-platform. Converts OpenSSH keys internally, no separate tool needed. |
| Mac | **Terminal (`sftp`)** | Built-in | No install needed. Command-line only. |
| Mac | **Cyberduck** | Free | GUI, drag-and-drop. |
| Mac | **Termius** | Free/Paid | Modern SSH/SFTP client, also on iOS/Android. |
| Mac | **Transmit** | Paid | Polished native Mac client. |

> MobaXterm is **Windows-only** and has no native Mac build. Closest Mac equivalents: Termius or Cyberduck.

---

## 4. Prerequisites

- AWS account with permissions to create S3 buckets, IAM roles/policies, and Transfer Family servers.
- 30–40 minutes.
- Windows: MobaXterm, WinSCP, or FileZilla installed.
- Mac: nothing extra needed for Terminal; otherwise install Cyberduck, Termius, FileZilla, or Transmit.

> **Cost note:** The SFTP server bills hourly from creation until deletion — there is no "stop" option. Delete it when done (Section 11).

---

## 5. Step 1 — Generate an SSH Key Pair

The **public** key goes into AWS; the **private** key stays on your machine.

### On Mac (Terminal)
```
ssh-keygen -t rsa -b 4096 -f ~/.ssh/transfer-family-lab -N ""
```
This works correctly on macOS/Linux shells — no gotchas.

View the public key to copy it later:
```
cat ~/.ssh/transfer-family-lab.pub
```

### On Windows (PowerShell)

> ⚠️ **Known issue:** Do **not** add `-N ""` to this command in PowerShell. PowerShell's argument quoting does not pass an empty string through correctly — it can silently set your key's passphrase to the literal two-character string `""` instead of leaving it blank. This causes a confusing `Permission denied (publickey)` error later, even though the key looks fine.

Run it **without** `-N`:
```
ssh-keygen -t rsa -b 4096 -f $env:USERPROFILE\.ssh\transfer-family-lab
```
When prompted:
```
Enter passphrase (empty for no passphrase):
```
Press **Enter** (leave blank), then press **Enter** again to confirm.

View the public key:
```
type $env:USERPROFILE\.ssh\transfer-family-lab.pub
```

Copy the full output (starts with `ssh-rsa ...`) — you'll paste it into AWS in Step 4.

---

## 6. Step 2 — Create the S3 Bucket

1. AWS Console → **S3** → **Create bucket**.
2. Bucket name: pick something globally unique, e.g. `your-name-transfer-family-lab`.
3. Leave **Block Public Access** turned **ON** (default).
4. Leave **Default encryption** at the default (SSE-S3 / Amazon S3 managed keys) unless you have a specific reason for SSE-KMS — a customer-managed KMS key requires extra `kms:GenerateDataKey`/`kms:Decrypt` permissions on the IAM role, which adds complexity for no benefit in this lab.
5. Click **Create bucket**.

> ⚠️ **Write down your exact bucket name now.** The single most common failure in this whole lab is a bucket-name mismatch between what you type in the S3 console and what you paste into the IAM policy in the next step. Copy-paste it, don't retype it.

---

## 7. Step 3 — Create the IAM Policy and Role

### 7.1 Create the policy

1. IAM → **Policies** → **Create policy** → **JSON** tab.
2. Paste this, replacing **every instance** of `your-bucket-name` with your **exact** bucket name from Step 2:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": ["s3:ListBucket"],
         "Resource": "arn:aws:s3:::your-bucket-name"
       },
       {
         "Effect": "Allow",
         "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject", "s3:GetObjectVersion"],
         "Resource": "arn:aws:s3:::your-bucket-name/*"
       }
     ]
   }
   ```
   > ⚠️ **This is the #1 real-world failure point.** There are **two** `Resource` lines above — one for the bucket itself (no trailing `/*`, used by `ListBucket`) and one for objects inside it (with `/*`, used by `PutObject`/`GetObject`/etc). It's very easy to update only one of the two and leave a placeholder or wrong bucket name in the other. Before saving, re-read both `Resource` lines side by side and confirm they both say your real bucket name.
3. Name it, e.g. `transfer-lab-policy`, and create it.

### 7.2 Create the role

1. IAM → **Roles** → **Create role** → **Custom trust policy**, paste:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": { "Service": "transfer.amazonaws.com" },
         "Action": "sts:AssumeRole"
       }
     ]
   }
   ```
2. Attach the `transfer-lab-policy` you just created.
3. Name the role, e.g. `transfer-lab-role`, and create it.

---

## 8. Step 4 — Create the Transfer Family Server

1. AWS Console → **AWS Transfer Family** → **Servers** → **Create server**.
2. **Choose protocols** → select **SFTP** → **Next**.
3. **Choose an identity provider** → select **Service managed** → **Next**.
4. **Choose an endpoint** → select **Publicly accessible** → **Next**.
5. **Choose a domain** → select **Amazon S3** → **Next**.
6. **Configure additional details** — leave everything at default/blank for this lab:
   - Managed workflows: leave both dropdowns blank (not needed).
   - Logging: leave "Create a new log group" selected — **keep this on**, it's how you'll debug any access-denied errors later (see Section 10).
   - Security policy, Server host key, Tags, Optimised directories: leave at default.
   - Click **Next**.
7. **Review and create** — confirm everything matches what you selected, then click **Create server**.
8. Wait for the server's **Status** to show **Online** (a few minutes — refresh the page).

---

## 9. Step 5 — Create the SFTP User

1. Open your server → **Users** tab → **Add user**.
2. **Username**: e.g. `labuser`.
3. **Role**: select `transfer-lab-role`.
4. **Policy**: select **"Select a policy from IAM"** and choose `transfer-lab-policy` (this attaches your session policy — it must exist and be correct, per Section 7.1).
5. **Home directory**: select your bucket from the dropdown (don't type it manually — picking from the dropdown avoids typos).
6. Leave **Restricted** unchecked unless you specifically want to lock the user to only see their home folder.
7. **SSH public key**: paste the full public key from Step 1.
8. Click **Add user**.

Copy the server's **Endpoint** value from the server detail page:
```
s-xxxxxxxxxxxxxxxxx.server.transfer.us-east-1.amazonaws.com
```

---

## 10. Step 6 — Connecting and Transferring Files

### On Mac — Terminal (built-in)

Connect:
```
sftp -i ~/.ssh/transfer-family-lab labuser@s-xxxxxxxxxxxxxxxxx.server.transfer.us-east-1.amazonaws.com
```
Type `yes` if prompted about host authenticity.

**Upload any file (text, image, PDF, zip — SFTP transfers everything in binary automatically):**
```
put photo.jpg
```
```
put text-file-demo.txt
```

**Download any file:**
```
get photo.jpg
```
Download with a different local name:
```
get photo.jpg renamed-photo.jpg
```
Download to a specific folder:
```
get photo.jpg ~/Downloads/photo.jpg
```
Download an entire folder recursively:
```
get -r foldername
```

**List what's in the bucket:**
```
ls
```

**Filenames with spaces — wrap in quotes:**
```
get "my photo.jpg"
```

Exit the session:
```
exit
```

> ⚠️ **Important — policy changes require a fresh connection.** Transfer Family snapshots your IAM session policy at the moment you connect. If you edit the IAM policy *while already connected*, the running session keeps using the **old** policy until you disconnect and reconnect. If you fix a permissions issue and it still fails, always `exit` and start a brand-new `sftp` connection before retesting.

### On Mac — GUI clients (Cyberduck / Termius / FileZilla)

Same idea, drag-and-drop instead of commands:
- **Cyberduck**: Open Connection → SFTP → paste endpoint, port 22, username, choose your private key → Connect → drag files in/out.
- **Termius**: New Host → paste endpoint, username → import your private key under Keys → connect → use its SFTP panel.
- **FileZilla**: Site Manager → New Site → SFTP → paste endpoint → Logon Type: Key file → browse to your private key (auto-converts internally) → Connect → drag files between local/remote panes.

### On Windows

- **MobaXterm**: Session → SFTP → paste endpoint, port 22, username → Advanced SFTP settings → Use private key → browse to your key file (OpenSSH format accepted directly) → connect → drag-and-drop in the SFTP panel.
- **WinSCP**: New Site → SFTP → paste endpoint, port 22, username → Advanced → SSH → Authentication → browse to private key (accept the `.ppk` conversion prompt) → Login → drag-and-drop.
- **FileZilla**: identical to the Mac steps above.

### Downloading directly via the AWS Console (no SFTP client needed)

If you just want to grab a file without going through SFTP at all:
1. S3 Console → your bucket → click the file.
2. Click **Download**.

---

## 11. Verifying the Transfer Worked

1. S3 Console → your bucket → confirm the uploaded file appears.
2. If you downloaded a file, open it locally to confirm it's intact (e.g., `open photo.jpg` on Mac).

---

## 12. Troubleshooting (Real Issues Hit During Testing, With Fixes)

| Symptom | Actual Cause Found | Fix |
|---|---|---|
| `Permission denied (publickey)` | Public key not attached to the user, wrong private key path in client, or (Windows) `-N ""` set the passphrase to the literal string `""` | Re-check Step 5's SSH key; on Windows, regenerate the key without `-N` and press Enter twice at the prompts |
| `dest open "...": Permission denied` on `put` | IAM policy had a leftover **placeholder bucket name** in the `ListBucket` statement's `Resource` line, while the second statement had the correct bucket — only one of the two `Resource` lines was updated | Open the policy JSON and check **both** `Resource` lines match your real bucket name exactly (see the warning box in Section 7.1) |
| Fixed the IAM policy but `put` **still** fails with `Access denied` | The active SFTP session had already cached the **old** policy from before the fix — confirmed by checking the CloudWatch log's `user-policy` field, which still showed the stale JSON | `exit` the SFTP session completely and reconnect fresh — policy changes only apply to new sessions, not ones already connected |
| `sudo put file.txt` → `Invalid command` | SFTP has no `sudo` concept — it's not a shell, so any "permission denied" here is always IAM/S3-side, never a client-side privilege issue | Never try `sudo` in an `sftp>` prompt; diagnose via IAM policy and CloudWatch logs instead |
| `Connection timed out` | Server still provisioning, or endpoint type isn't Publicly accessible | Wait for server status **Online**; confirm endpoint type from Step 4 |
| Files upload but don't appear in the bucket you're checking | Wrong bucket selected as the user's home directory | Re-check Step 5's home directory dropdown selection |
| WinSCP prompts to convert the key | Normal — WinSCP converts OpenSSH keys to `.ppk` on first use (MobaXterm and FileZilla don't need this) | Accept the conversion prompt and save the `.ppk` copy |
| `ssh-keygen`: file already exists | A key already exists at that path from a previous attempt | Choose a new filename, or delete the old key pair first |
| MobaXterm won't run on Mac | MobaXterm is Windows-only software | Use Termius or Cyberduck instead |

### How to get the exact denial reason yourself (don't just guess)

If something is still denied after checking the table above, go straight to the source of truth:

1. **CloudWatch → Log groups** → find `/aws/transfer/<your-server-id>`.
2. Open the latest log stream.
3. Find the `CONNECTED` entry — it includes a `user-policy` field showing **exactly** which IAM policy JSON was active for that session. Compare it character-for-character against what you think you saved in IAM.
4. Find the `ERROR` entry for your failed operation — it shows the `path`, `mode` (e.g., `CREATE|TRUNCATE|WRITE` for uploads), and `message` (e.g., `Access denied`). This tells you precisely which file path was denied.

This log-based check is far faster than guessing — it's how the placeholder-bucket-name bug and the stale-session-policy bug above were actually found and fixed.

---

## 13. Cleanup (Stop Billing)

Delete in this order:

1. **Transfer Family server** → Servers → select it → **Delete**.
2. **IAM role** → Roles → delete `transfer-lab-role` (detach the policy first if prompted).
3. **IAM policy** → Policies → delete `transfer-lab-policy`.
4. **S3 bucket** → empty all objects/versions first, then delete the bucket.
5. *(Optional)* remove local key pairs:
   - Mac: `rm ~/.ssh/transfer-family-lab ~/.ssh/transfer-family-lab.pub`
   - Windows (PowerShell): `Remove-Item $env:USERPROFILE\.ssh\transfer-family-lab*`

> The server bills hourly from creation until deletion — delete it as soon as you're done.

---
