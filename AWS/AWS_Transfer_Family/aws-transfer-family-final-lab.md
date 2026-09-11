# AWS Transfer Family — SFTP Server: Complete End-to-End Lab (Windows + Mac Clients)

> Verified, full hands-on guide: create an SFTP server backed by S3 in the AWS Console, then connect and transfer files using client applications on **both Windows and Mac**. No steps skipped.

---

## 1. Scenario

You will build one AWS Transfer Family SFTP server backed by S3, then test connecting to it from **two different operating systems** using the right tools for each:

- **Windows** → GUI client (MobaXterm, WinSCP, or FileZilla)
- **Mac** → built-in Terminal (`sftp` command) or a GUI client (Cyberduck, Termius, FileZilla, Transmit)

The AWS-side setup is identical regardless of which OS/client you use — only the connection method on your local machine changes.

---

## 2. Architecture

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/3df900b1-4491-493d-a610-f73daaf25955" />


---

## 3. Client Applications — Which One for Which OS

### Windows options

| Application | Type | Notes |
|---|---|---|
| **MobaXterm** | Free (Home Edition) / Paid Pro | Combines terminal, SFTP file browser, and SSH client in one window. Windows-only — does **not** run natively on Mac. |
| **WinSCP** | Free | Dedicated GUI SFTP/FTP client, drag-and-drop, very beginner-friendly. |
| **FileZilla** | Free | Cross-platform (also on Mac/Linux), classic drag-and-drop SFTP/FTP client. |

### Mac options

| Application | Type | Notes |
|---|---|---|
| **Terminal (`sftp` command)** | Built-in, free | No install needed — already on every Mac. Command-line only. |
| **Cyberduck** | Free | GUI, drag-and-drop, supports SFTP and S3 directly. |
| **Termius** | Free tier / Paid Pro | Modern SSH/SFTP client, also available on iOS/Android. |
| **FileZilla** | Free | Same app as Windows — cross-platform. |
| **Transmit** | Paid | Polished, native Mac look and feel. |

> MobaXterm is Windows-only. The closest Mac equivalents are **Termius** or **Cyberduck**.

---

## 4. Prerequisites

- An AWS account with permissions to create S3 buckets, IAM roles/policies, and Transfer Family servers.
- Roughly 30–40 minutes.
- Windows: MobaXterm, WinSCP, or FileZilla installed (pick one).
- Mac: nothing to install if using Terminal; otherwise Cyberduck, Termius, FileZilla, or Transmit.

> **Cost note:** The SFTP server bills hourly from creation until deletion, regardless of usage — there is no "stop" option for service-managed SFTP servers. Do the cleanup step (Section 9) once you're done.

---

## 5. AWS Console Setup (Same for Every Client — Do This First)

### Step 1 — Generate an SSH key pair

The **public** key gets uploaded to AWS; the **private** key stays on your machine for authentication.

**On Mac (Terminal):**
```
ssh-keygen -t rsa -b 4096 -f ~/.ssh/transfer-family-lab -N ""
```
This works fine on macOS/Linux shells. View the public key:
```
cat ~/.ssh/transfer-family-lab.pub
```

**On Windows (PowerShell):**
```
ssh-keygen -t rsa -b 4096 -f $env:USERPROFILE\.ssh\transfer-family-lab
```
> **Do not** try to pass `-N ""` in PowerShell to set an empty passphrase — PowerShell's argument quoting does not pass an empty string through correctly to `ssh-keygen.exe`, and it can silently set your passphrase to the literal two-character string `""` instead of leaving it empty. This causes confusing `Permission denied (publickey)` errors later even though the key looks correct.
>
> Instead, just run the command above **without** `-N`, and when prompted:
> ```
> Enter passphrase (empty for no passphrase):
> ```
> press **Enter** (leave it blank), then press **Enter** again to confirm.

View the public key:
```
type $env:USERPROFILE\.ssh\transfer-family-lab.pub
```

Copy the full public key output (starts with `ssh-rsa ...`) — you'll paste it into the AWS console in Step 5.

### Step 2 — Create the S3 bucket

1. AWS Console → **S3** → **Create bucket**.
2. Bucket name: something globally unique, e.g. `your-name-transfer-family-lab`.
3. Leave **Block Public Access** turned **ON** (default) — the bucket does not need to be public.
4. Leave all other settings at default.
5. Click **Create bucket**.

### Step 3 — Create the IAM policy and role

1. AWS Console → **IAM** → **Policies** → **Create policy**.
2. Switch to the **JSON** tab and paste (edit the bucket name to match yours):
   ```
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": ["s3:ListBucket"],
         "Resource": "arn:aws:s3:::your-name-transfer-family-lab"
       },
       {
         "Effect": "Allow",
         "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject", "s3:GetObjectVersion"],
         "Resource": "arn:aws:s3:::your-name-transfer-family-lab/*"
       }
     ]
   }
   ```
3. Name it, e.g. `transfer-family-lab-policy`, and create it.
4. Go to **IAM** → **Roles** → **Create role**.
5. Choose **Custom trust policy** and paste:
   ```
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
6. On the next screen, attach the `transfer-family-lab-policy` you just created.
7. Name the role, e.g. `transfer-family-lab-role`, and click **Create role**.

### Step 4 — Create the Transfer Family server

1. AWS Console → **AWS Transfer Family** → **Servers** → **Create server**.
2. **Choose protocols**: select **SFTP**, click **Next**.
3. **Choose an identity provider**: select **Service managed**, click **Next**.
4. **Choose an endpoint**: select **Publicly accessible**, click **Next**.
   *(This creates a PUBLIC endpoint type — simplest for this lab. VPC-hosted is the production-recommended option for tighter network control but adds VPC/security-group steps.)*
5. On the storage/domain step, choose **Amazon S3** as the domain.
6. Leave logging role and security policy at defaults for the lab, then **Create server**.
7. Wait until the server's **Status** shows **Online** (can take a few minutes — refresh the page).

### Step 5 — Create the SFTP user

1. Open your server → **Users** tab → **Add user**.
2. **Username**: e.g. `labuser`.
3. **Role**: select `transfer-family-lab-role`.
4. **Home directory**: select your bucket, e.g. `your-name-transfer-family-lab`.
5. **SSH public key**: paste the full public key you copied in Step 1.
6. Click **Add user**.

### Step 6 — Copy the server endpoint

On the server's detail page, copy the **Endpoint** value:
```
s-xxxxxxxxxxxxxxxxx.server.transfer.us-east-1.amazonaws.com
```
You will use this as the "Host" / "Remote host" in every client below.

---

## 6. Connecting from Mac

### Option A — Terminal (built-in, no install)

```
sftp -i ~/.ssh/transfer-family-lab labuser@s-xxxxxxxxxxxxxxxxx.server.transfer.us-east-1.amazonaws.com
```
Type `yes` if prompted about host authenticity.

Upload a test file:
```
echo "hello from my mac" > test-file.txt
```
```
put test-file.txt
```
Download it back:
```
get test-file.txt downloaded-test-file.txt
```
Exit:
```
exit
```

### Option B — Cyberduck (GUI)

1. Open Cyberduck → **Open Connection**.
2. Choose **SFTP (SSH File Transfer Protocol)** from the dropdown.
3. **Server**: paste your endpoint from Step 6.
4. **Port**: 22.
5. **Username**: `labuser`.
6. Click the **SSH Private Key** dropdown → **Choose** → select `~/.ssh/transfer-family-lab`.
7. Click **Connect**.
8. Drag and drop a file from Finder into the Cyberduck window to upload; drag a file out to download.

### Option C — Termius (GUI)

1. Open Termius → **+ New Host**.
2. **Address**: paste your endpoint from Step 6.
3. **Username**: `labuser`.
4. Under **Keys**, click **+ New Key** → **Import from file** → select `~/.ssh/transfer-family-lab`.
5. Save, then click the host to connect. Use Termius's built-in SFTP panel to drag-and-drop files.

### Option D — FileZilla (GUI)

1. Open FileZilla → **File** → **Site Manager** → **New Site**.
2. **Protocol**: SFTP.
3. **Host**: paste your endpoint from Step 6.
4. **Logon Type**: Key file.
5. **User**: `labuser`.
6. Browse to `~/.ssh/transfer-family-lab` as the key file (FileZilla converts OpenSSH-format keys internally — no separate PuTTYgen step needed).
7. Click **Connect**.
8. Drag and drop files between the local pane (left) and remote pane (right).

---

## 7. Connecting from Windows

### Option A — MobaXterm (GUI + terminal combo)

1. Open MobaXterm → **Session** → **SFTP**.
2. **Remote host**: paste your endpoint from Step 6.
3. **Port**: 22.
4. **Username**: `labuser`.
5. Under **Advanced SFTP settings**, check **Use private key**, then browse to your private key file (`transfer-family-lab` — MobaXterm accepts OpenSSH-format keys directly, no conversion needed).
6. Click **OK** — the session opens with a terminal on top and a drag-and-drop SFTP file browser panel on the left.
7. Drag a file from Windows Explorer into the left SFTP panel to upload; right-click a remote file → **Download** to pull it back.

### Option B — WinSCP (GUI)

1. Open WinSCP → **New Site**.
2. **File protocol**: SFTP.
3. **Host name**: paste your endpoint from Step 6.
4. **Port number**: 22.
5. **User name**: `labuser`.
6. Click **Advanced** → **SSH** → **Authentication** → browse to **Private key file** → select your key. WinSCP will offer to convert your OpenSSH key to `.ppk` format the first time — accept and save the converted copy.
7. Click **OK**, then **Login**.
8. Drag and drop files between the local pane (left) and remote pane (right).

### Option C — FileZilla (GUI)

Same steps as **Mac FileZilla** in Section 6, Option D — FileZilla's interface and key handling are identical across Windows and Mac.

---

## 8. Verifying the Transfer Worked

Regardless of which client/OS you used:
1. Go to the **S3 console** → open your bucket.
2. Confirm `test-file.txt` (or whatever you uploaded) appears there.
3. If you downloaded a file back, confirm it opened correctly and matches the original content.

---

## 9. Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `Permission denied (publickey)` | Public key not attached to the user, wrong private key selected, or (on Windows) an empty passphrase that accidentally got set to `""` | Re-check Step 5; confirm the client points at the correct private key; on Windows, regenerate the key without `-N` and press Enter twice when prompted, as noted in Step 1 |
| `Connection timed out` | Server still provisioning, or endpoint isn't Publicly accessible | Wait for server status **Online**; confirm endpoint type from Step 4 |
| `Access Denied` on upload | IAM policy doesn't grant `s3:PutObject` on the correct bucket ARN | Re-check the IAM policy resource ARNs match your exact bucket name (Step 3) |
| Files upload but don't appear in S3 console | Wrong bucket selected as the user's home directory | Re-check Step 5's home directory setting |
| WinSCP asks to convert key format | Normal — WinSCP converts OpenSSH keys to `.ppk` on first use (MobaXterm and FileZilla don't require this) | Accept the conversion prompt and save the `.ppk` copy |
| `ssh-keygen` "file already exists" prompt | A key already exists at that path from a previous attempt | Choose a new filename, or delete the old key pair first |
| MobaXterm won't install/open on Mac | MobaXterm is Windows-only software | Use Termius or Cyberduck instead — see Section 6 |

---

## 10. Cleanup (Do This to Stop Charges)

Delete resources in this order:

1. **Transfer Family server** → Servers → select it → **Delete**.
2. **IAM role** → Roles → delete `transfer-family-lab-role` (detach the policy first if prompted).
3. **IAM policy** → Policies → delete `transfer-family-lab-policy`.
4. **S3 bucket** → empty all objects/versions first, then delete the bucket.
5. *(Optional)* remove local key pairs:
   - Mac: `rm ~/.ssh/transfer-family-lab ~/.ssh/transfer-family-lab.pub`
   - Windows (PowerShell): `Remove-Item $env:USERPROFILE\.ssh\transfer-family-lab*`

> The SFTP server bills hourly from creation until deletion — delete it as soon as you're finished testing.

---

*End-to-end lab complete and verified against current AWS documentation and client-app behavior — AWS Console setup done once, tested from both Windows and Mac clients.*
