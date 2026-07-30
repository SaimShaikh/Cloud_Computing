# AWS Transform MGN — End-to-End Migration Lab
### Web App + Custom Domain (Hostinger) + SSL/TLS + Full Cutover
### Mumbai (ap-south-1) → Singapore (ap-southeast-1)

**Account type:** Personal AWS account (POC)
**Goal:** Deploy a real web app on Ubuntu 22.04, attach a Hostinger-registered domain, secure it with a real Let's Encrypt certificate, replicate it to Singapore with AWS Transform MGN, validate it with a Test Launch, and perform a proper production Cutover that includes the DNS switch — with zero shortcuts.

> **Rebrand notice:** AWS Application Migration Service (AWS MGN) has been rebranded **AWS Transform MGN**, accessed through the AWS Transform console rather than a standalone "Application Migration Service" search result. This is branding and navigation only — same replication engine, same APIs, same IAM policies, same ports, same lifecycle states, same compliance certifications (FedRAMP High, HIPAA, PCI DSS, ISO, SOC 1/2/3). If your console search shows "AWS Transform" instead of "Application Migration Service," that's expected — every step below reflects the current navigation.

This supersedes the earlier drafts. Every step below has been re-verified for correctness and sequencing, and these gaps from previous versions are fixed:
1. No Elastic IP → a domain can't reliably point at an EC2 instance whose public IP changes on stop/start.
2. No real domain/SSL flow → self-signed certs don't reflect how a production cutover with DNS actually works.
3. No DNS-cutover procedure → migrating the server isn't the same as migrating traffic; the domain switch is a distinct, ordered step.
4. Console navigation updated to reflect the AWS Transform rebrand.

---

## 1. Architecture

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/c8f83680-5075-4e04-bd15-e0c41cce0559" />


**Key principle this lab teaches:** migrating the *server* (MGN's job) and migrating the *traffic* (DNS cutover) are two separate, sequential operations. The server migration can be tested safely without affecting live traffic; the DNS switch is the one moment that actually redirects users.

---

## 2. Prerequisites

| Item | Detail |
|---|---|
| Domain | A domain registered/managed in Hostinger, with access to hPanel → DNS Zone Editor |
| Email | A real email for Let's Encrypt registration/expiry notices |
| AWS account | Same personal account, both regions |
| IAM | Your IAM user/role needs `AWSApplicationMigrationFullAccess` + `AWSApplicationMigrationEC2Access` |
| Elastic IPs | You will allocate two — one per region (small ongoing cost only if left unassociated) |

---

## Phase 1 — Build and Secure the Source Server (Mumbai)

### Task 1.1 — Launch the source EC2 instance
EC2 → Launch instance (region = **ap-south-1**)

| Setting | Value |
|---|---|
| AMI | Ubuntu Server 22.04 LTS |
| Instance type | t3.micro |
| Storage | 30 GB gp3 |
| Security Group | 22 (SSH, your IP only), 80 (HTTP, 0.0.0.0/0), 443 (HTTPS, 0.0.0.0/0) |

### Task 1.2 — Allocate and associate an Elastic IP
EC2 → **Elastic IPs** → Allocate → Associate with the instance.

> Correction from the earlier draft: a plain EC2 public IP is not stable — it changes if the instance stops/starts. Since you're pointing a real domain at this server, you need an Elastic IP (call it **EIP-A**) so DNS has a fixed target.

**Verify:** `curl -4 ifconfig.me` on the instance returns EIP-A.

### Task 1.3 — Update OS and set hostname
```bash
sudo apt update && sudo apt upgrade -y
sudo hostnamectl set-hostname mumbai-web-server
```
**Verify:** `hostname` → `mumbai-web-server`

### Task 1.4 — Install Nginx and Python tooling
```bash
sudo apt install nginx python3-venv python3-pip -y
sudo systemctl enable nginx
sudo systemctl start nginx
```
**Verify:** `systemctl status nginx` → `active (running)`

### Task 1.5 — Deploy the web app
A small Flask app run under Gunicorn, managed by systemd, reverse-proxied by Nginx — a realistic, production-shaped pattern.

```bash
sudo mkdir -p /opt/webapp
sudo python3 -m venv /opt/webapp/venv
sudo /opt/webapp/venv/bin/pip install flask gunicorn
```

Create `/opt/webapp/app.py`:
```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def home():
    return "<h1>Hello from mumbai-web-server</h1><p>AWS MGN migration demo app.</p>"

@app.route("/health")
def health():
    return {"status": "ok", "host": "mumbai-web-server"}
```

Create the systemd unit `/etc/systemd/system/webapp.service`:
```ini
[Unit]
Description=Gunicorn instance for demo web app
After=network.target

[Service]
User=www-data
WorkingDirectory=/opt/webapp
ExecStart=/opt/webapp/venv/bin/gunicorn --workers 2 --bind 127.0.0.1:5000 app:app
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo chown -R www-data:www-data /opt/webapp
sudo systemctl daemon-reload
sudo systemctl enable --now webapp.service
sudo systemctl status webapp.service
```
**Verify:** `curl http://127.0.0.1:5000/health` → `{"status": "ok", ...}`

### Task 1.6 — Configure Nginx as a reverse proxy
Create `/etc/nginx/sites-available/webapp`:
```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
Replace `yourdomain.com` with your actual domain throughout this lab.

```bash
sudo ln -s /etc/nginx/sites-available/webapp /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```
> Correction carried over from the previous version: removing `sites-enabled/default` is mandatory — otherwise Nginx may serve the default page instead of your `server_name`-matched vhost depending on request order.

**Verify:** `curl http://<EIP-A>` → `Hello from mumbai-web-server`

### Task 1.7 — Point the Hostinger domain at the source
Hostinger → **hPanel** → Domains → select your domain → **DNS / Nameservers** → **DNS Zone Editor**.

Add or edit:

| Type | Name | Points to | TTL |
|---|---|---|---|
| A | `@` | `<EIP-A>` | 300 (5 min) |
| A | `www` | `<EIP-A>` | 300 |

> Set TTL as low as Hostinger allows now (not later). A low TTL is what makes the eventual cutover DNS switch propagate in minutes instead of hours — this only works if it's already low *before* you need to switch.

**Verify propagation** (can take a few minutes):
```bash
dig +short yourdomain.com
dig +short www.yourdomain.com
```
Both should return `EIP-A`. Confirm from a public resolver too: `dig +short yourdomain.com @8.8.8.8`.

Do not proceed to Task 1.8 until this resolves correctly — Let's Encrypt's HTTP-01 challenge validates by connecting to your domain, so DNS must already point to this server.

### Task 1.8 — Obtain a real SSL/TLS certificate (Let's Encrypt via Certbot)
```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com \
  --email you@example.com --agree-tos --no-eff-email
```
Certbot will:
- Validate domain ownership via an HTTP-01 challenge (a temporary file served over port 80)
- Obtain the certificate from Let's Encrypt
- Automatically edit the Nginx vhost to add a `listen 443 ssl` block and an HTTP→HTTPS redirect

**Verify:**
```bash
sudo nginx -t
curl -Ik https://yourdomain.com
```
Expected: `HTTP/2 200`, and the certificate issuer is Let's Encrypt (check in browser padlock too).

### Task 1.9 — Confirm auto-renewal is active
```bash
systemctl list-timers | grep certbot
sudo certbot renew --dry-run
```
Expected: a `certbot.timer` (or `snap.certbot.renew.timer`) listed, and the dry run reports success.

> This matters directly for migration: `/etc/letsencrypt/` (certificate, private key, and renewal config) lives on disk, so MGN's block-level replication carries it to Singapore automatically. You are not reissuing a cert from scratch after cutover — you're carrying forward a working one and confirming renewal still works from the new location.

### Task 1.10 — Add verification test data (optional but recommended)
```bash
sudo mkdir -p /opt/migration-demo/{documents,logs,backups}
echo "Migration Test" | sudo tee /opt/migration-demo/documents/info.txt
sudo fallocate -l 2G /opt/migration-demo/backups/data1.img
crontab -e
```
Add:
```
*/5 * * * * echo "Cron running $(date)" >> /opt/migration-demo/logs/cron.log
```
This gives you independent, easy-to-check proof (beyond "the website loads") that block-level data, not just the OS, replicated correctly.

### Task 1.11 — Pre-migration checklist

| Check | Command | Expected |
|---|---|---|
| Hostname | `hostname` | `mumbai-web-server` |
| App service | `systemctl status webapp` | `active (running)` |
| Nginx config | `sudo nginx -t` | `syntax is ok` |
| HTTP→HTTPS redirect | `curl -I http://yourdomain.com` | `301` to `https://` |
| HTTPS site | `curl -Ik https://yourdomain.com` | `200`, valid Let's Encrypt cert |
| App health | `curl https://yourdomain.com/health` | `{"status":"ok",...}` |
| Cert auto-renew | `sudo certbot renew --dry-run` | Success |
| DNS | `dig +short yourdomain.com` | `EIP-A` |
| Cron | `cat /opt/migration-demo/logs/cron.log` | Updating every 5 min |

Do not proceed to Phase 2 until every row passes.

---

## Phase 2 — Configure AWS Transform MGN (Singapore)

### Task 2.1 — Switch console region
Top-right region selector → **Asia Pacific (Singapore) / ap-southeast-1**. MGN is always initialized and operated in the **target** region.

### Task 2.2 — Open AWS Transform MGN and initialize
Console search bar → type **"Transform"** or **"MGN"** → open **AWS Transform**.

On the AWS Transform landing page (the "Migration and modernisation" screen with the **Get started** panel), the MGN engine is what powers the **Migrate Workloads** capability. To reach the classic MGN source-servers workflow directly:
- Use the console search bar and search **"Transform MGN"**, or
- From the AWS Transform landing page, choose the **Migrate Workloads** card under "With AWS Transform you can."

The first time you open it, you'll be prompted to initialize the service — this is the same one-time step as before, just under the new name. Choose **Get started**. This creates the service-linked role `AWSServiceRoleForApplicationMigrationService`, a default replication template, and a default launch template. Wait for the **Source servers** dashboard (not the setup wizard) before continuing.

> Note: The "Enable web application" button on the AWS Transform landing page is for AWS Transform's own agentic assessment/modernization web app (used for things like mainframe and VMware portfolio analysis) — it is a separate onboarding gate from the MGN rehost engine and is **not required** for this lab. You only need the Transform MGN / Migrate Workloads path.

### Task 2.3 — Configure the Replication Settings Template
AWS Transform MGN → **Settings** → **Replication settings template**

| Setting | Value |
|---|---|
| Replication server instance type | t3.small |
| Staging disk type | gp3 |
| Bandwidth throttling | Disabled (POC) |
| Staging area subnet | A subnet in your Singapore VPC |
| Security group | Dedicated `mgn-replication-sg` (create in Task 2.4) |
| Data routing | **Public IP** — correct for a single-account POC with two independent regional VPCs and no VPC peering/Transit Gateway/VPN between them |

> Security note: choosing Public IP routing does not mean replication traffic is unencrypted. AWS Transform MGN uses TLS on both the control plane (agent registration/API calls, TCP 443) and the data plane (block replication, TCP 1500), regardless of whether Public or Private IP routing is selected.

### Task 2.4 — Create the replication security group
Singapore VPC → create `mgn-replication-sg`:

| Type | Port | Source |
|---|---|---|
| TCP | 1500 | Your source server's Elastic IP (EIP-A) |

### Task 2.5 — Set up IAM for the replication agent
The agent needs its own authenticated identity — separate from MGN's service-linked role.

**Recommended (EC2-to-EC2, no long-lived keys):**
1. IAM → Roles → Create role → attach the AWS managed policy `AWSApplicationMigrationServiceEc2InstancePolicy`.
2. Attach this role as an instance profile to the **Mumbai source instance** (EC2 → Actions → Security → Modify IAM role).
3. The installer will use the instance's credentials automatically — no keys needed.

### Task 2.6 — Add the source server
AWS Transform MGN → **Source servers** → **Add source server** → OS: **Linux**. Copy the installer command shown — it is unique to your account/region. Do not reuse a command from documentation.

### Task 2.7 — Pre-install checks on Mumbai
```bash
python3 --version
curl --version
wget --version
systemctl status nginx webapp
```

### Task 2.8 — Install the replication agent
```bash
sudo python3 aws-replication-installer-init.py --region ap-southeast-1 --no-prompt
```
(No access key arguments needed with the instance profile from Task 2.5.)

### Task 2.9 — Monitor replication
AWS Transform MGN → Source servers → `mumbai-web-server`. Two independent status columns:

| Data replication status | Meaning |
|---|---|
| In progress | Initial sync running |
| Healthy | Continuous replication, low lag |

| Lifecycle state | Meaning |
|---|---|
| Not ready | Initial sync incomplete |
| **Ready for testing** | Gate for Phase 3 |

Wait for lifecycle state = **Ready for testing** before continuing — this is the real gate, not "Healthy" data status alone.

---

## Phase 3 — Launch Template and Test Launch

### Task 3.1 — Configure the Launch Template
AWS Transform MGN → Source servers → `mumbai-web-server` → **Launch settings**

| Setting | Value |
|---|---|
| Instance type | t3.micro |
| Subnet | Singapore target subnet |
| Security group | Allow 22, 80, 443 |
| Public IP | Enabled (temporary, for testing only — the permanent address comes from an Elastic IP at cutover) |

### Task 3.2 — Launch a test instance
AWS Transform MGN → **Test and Cutover** → **Launch test instances**. This launches a disposable instance in Singapore from replicated data. It does **not** touch Mumbai or affect live traffic — DNS still points at EIP-A.

### Task 3.3 — Verify the test instance without touching DNS
Since the domain still resolves to Mumbai, use one of these to validate the Singapore instance directly by name:

**Option A — curl with a manual resolve override:**
```bash
curl --resolve yourdomain.com:443:<test-instance-public-ip> https://yourdomain.com/health
```
This forces curl to connect to the test instance's IP while still sending `yourdomain.com` in the TLS SNI and `Host` header — so both the correct Nginx vhost and the correct (replicated) certificate are exercised, exactly as they will be in production.

**Option B — temporary local hosts file entry** (on your own machine, not the server):
```
<test-instance-public-ip>  yourdomain.com  www.yourdomain.com
```
Then browse to `https://yourdomain.com` normally; remove the entry afterward.

**Verification checklist:**

| Check | Expected |
|---|---|
| `curl --resolve ... /health` | `{"status":"ok","host":"mumbai-web-server"}` (hostname unchanged) |
| Certificate served | Same Let's Encrypt cert as source (replicated, not reissued) |
| `systemctl status webapp` on test instance | `active (running)` |
| `ls /opt/migration-demo/backups` | `data1.img` present |
| `cat /opt/migration-demo/logs/cron.log` | Present, still updating |

### Task 3.4 — Finalize or extend the test
Pass → **Mark as ready for cutover** → **Finalize test** (terminates the test instance; Mumbai and live traffic are unaffected the entire time). Fail → troubleshoot (Section 7) and re-test.

---

## Phase 4 — Cutover Preparation

### Task 4.1 — Re-confirm DNS TTL is already low
Since it was set to 300s in Task 1.7, propagation after the real switch should complete within minutes. If you skipped that step, lower it now and **wait at least one full old-TTL period** before cutting over, so the old, higher TTL value fully expires from resolver caches first.

### Task 4.2 — Allocate the target Elastic IP
Singapore → EC2 → Elastic IPs → Allocate (call it **EIP-B**). Don't associate it yet — you'll associate it to the cutover instance in the next phase.

---

## Phase 5 — Cutover

### Task 5.1 — Launch the cutover instance
AWS Transform MGN → Source servers → `mumbai-web-server` → **Test and Cutover** → **Launch cutover instances**. Lifecycle state moves to "Cutover in progress" → "Cutover complete" once running.

### Task 5.2 — Associate the Elastic IP
EC2 → Elastic IPs → select **EIP-B** → **Associate** → choose the cutover instance.

> This is the step that gives the migrated server a permanent address before you touch DNS — without it, the cutover instance's IP could change later and silently break the site.

### Task 5.3 — Validate the cutover instance directly (before the DNS switch)
Repeat the Task 3.3 verification (`curl --resolve yourdomain.com:443:<EIP-B> https://yourdomain.com/health`) against **EIP-B**. Confirm app, data, and cert all check out — this is your last chance to catch a problem with zero user impact.

### Task 5.4 — Switch the domain (the actual cutover moment)
Hostinger → hPanel → DNS Zone Editor → edit both records:

| Type | Name | Points to |
|---|---|---|
| A | `@` | `EIP-B` |
| A | `www` | `EIP-B` |

### Task 5.5 — Verify propagation and live traffic
```bash
dig +short yourdomain.com @8.8.8.8
dig +short yourdomain.com @1.1.1.1
curl -Ik https://yourdomain.com
curl https://yourdomain.com/health
```
Expected: both resolvers return `EIP-B`, HTTPS responds `200` with a valid cert, and `/health` reports the app is up.

### Task 5.6 — Confirm certificate renewal works from the new location
```bash
sudo certbot renew --dry-run
```
Expected: success. The certificate itself didn't need to be reissued (it's bound to the domain, not the IP, and was carried over by replication), but renewal validation is domain-based, so this confirms the new server can renew it once DNS is fully live here.

### Task 5.7 — Finalize cutover
AWS Transform MGN console → **Finalize cutover**. This stops replication for this source server; the Mumbai instance itself is untouched by AWS Transform MGN and keeps running until you decide otherwise.

---

## Phase 6 — Post-Cutover Decommission (Mumbai)

### Task 6.1 — Keep Mumbai running for a grace period
Leave `mumbai-web-server` running for 24–48 hours after the DNS switch. Some resolvers cache beyond the TTL you set; this window catches any stragglers.

### Task 6.2 — Confirm no traffic is still hitting Mumbai
```bash
sudo tail -f /var/log/nginx/access.log
```
On the Mumbai instance — access should trail off to near zero once propagation is complete globally.

### Task 6.3 — Disconnect the agent and decommission
```bash
sudo systemctl stop aws-replication-agent
sudo systemctl disable aws-replication-agent
```
Then in the Singapore AWS Transform MGN console: Source servers → `mumbai-web-server` → **Disconnect from AWS**. Finally, terminate the Mumbai EC2 instance and release **EIP-A**.

---

## 7. Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Certbot HTTP-01 challenge fails | DNS not yet pointing at the server, or port 80 blocked | Re-check Task 1.7 `dig` output; confirm SG allows port 80 from `0.0.0.0/0` |
| Site shows default Nginx page | `sites-enabled/default` not removed, or `server_name` mismatch | Remove default site; confirm `server_name` matches the domain exactly |
| `curl --resolve` shows cert error on test instance | Replication hasn't finished copying `/etc/letsencrypt` yet | Wait for lifecycle "Ready for testing"; re-run the test launch |
| Domain doesn't resolve to EIP-B after cutover | DNS change not saved, or resolver cache | Re-check Hostinger DNS Zone Editor; test against multiple public resolvers; wait out TTL |
| `certbot renew --dry-run` fails post-cutover | DNS switch incomplete when the check ran, or SG on target blocks port 80 | Re-run after confirming `dig` returns `EIP-B` everywhere; check target SG |
| App works over HTTP but not HTTPS on the new instance | Certbot's Nginx edits didn't get carried into the reverse-proxy vhost correctly | `sudo nginx -t` for syntax errors; compare `/etc/nginx/sites-enabled/webapp` against the source |
| Website reachable but `/health` times out | `webapp.service` not running after replication/reboot | `systemctl status webapp`; check `journalctl -u webapp` for startup errors |
| Elastic IP not associated after cutover launch | Manual step (Task 5.2) skipped | Associate EIP-B to the cutover instance before switching DNS |
| Some users still hit Mumbai after cutover | TTL wasn't lowered far enough in advance | Expected during the old TTL's grace window; confirm it clears within that window |

---


## 8. Cheat Sheet

```bash
# Web app service
sudo systemctl status webapp
sudo journalctl -u webapp -f

# Nginx
sudo nginx -t && sudo systemctl reload nginx

# Certbot
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com --email you@example.com --agree-tos --no-eff-email
sudo certbot renew --dry-run
systemctl list-timers | grep certbot

# DNS checks
dig +short yourdomain.com @8.8.8.8
dig +short yourdomain.com @1.1.1.1

# Test a not-yet-live server by domain name
curl --resolve yourdomain.com:443:<target-ip> https://yourdomain.com/health

# AWS Transform MGN agent install (instance-profile auth, no keys)
sudo python3 aws-replication-installer-init.py --region ap-southeast-1 --no-prompt

# Disconnect agent before decommission
sudo systemctl stop aws-replication-agent
sudo systemctl disable aws-replication-agent
```



