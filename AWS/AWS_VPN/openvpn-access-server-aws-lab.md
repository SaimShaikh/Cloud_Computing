# OpenVPN Access Server on AWS — Complete Hands-on Lab Guide

**Level:** Intermediate | **Duration:** 60–90 minutes | **Cost:** ~$0.02–0.05/hr (t2.small) + OpenVPN AS licensing (2 free connections)

---

## 1. Scenario

You are a Cloud/DevOps engineer at a company that runs its workloads inside a private subnet with **no public IP addresses and no direct internet exposure**. Your team needs a way for engineers working remotely (from laptops, home networks, coffee shops) to securely reach these private EC2 instances for administration, debugging, and deployments — without opening SSH to the internet and without giving every engineer a bastion host key.

The chosen solution is **OpenVPN Access Server (AS)**, deployed from the AWS Marketplace as a self-managed VPN gateway. It sits in a public subnet, terminates VPN connections from client laptops, and routes authorized traffic into the private subnet where your workloads live.

By the end of this lab, you will have:

- A VPC with a public subnet (VPN gateway) and a private subnet (workload).
- An OpenVPN Access Server instance reachable from the internet on 443/TCP and 1194/UDP.
- A private EC2 instance with **no public IP**, reachable only through the VPN tunnel.
- A working end-to-end connection: laptop → OpenVPN Connect client → OpenVPN AS → private EC2 (SSH).

---

## 2. Concept Deep Dive

### 2.1 What OpenVPN Access Server actually is

OpenVPN AS is not just the open-source `openvpn` daemon — it's a full commercial-friendly VPN appliance that bundles:

- A **VPN daemon** (TCP 443 and UDP 1194) that terminates client tunnels.
- A **web-based Admin UI** (port 943) for configuring users, networks, and routing.
- A **Client Web UI** (port 943 or 443) where end users log in and download their `.ovpn` connection profile.
- Built-in **PKI (CA)** that issues certificates per user automatically — no manual OpenSSL cert generation needed.

This is why it's popular for hands-on labs and small/medium teams: you get a production-capable VPN without hand-rolling PKI and iptables rules yourself.

### 2.2 How this differs from the other VPN patterns in this series

| Pattern | Use case | Where it terminates | Client software |
|---|---|---|---|
| **Site-to-Site VPN** | Connect two networks (e.g. on-prem ↔ AWS) permanently | AWS Virtual Private Gateway / TGW | None — router/firewall to router/firewall |
| **AWS Client VPN** | Individual users connect to AWS VPC, fully managed by AWS | AWS-managed Client VPN endpoint | AWS-provided or any OpenVPN-compatible client |
| **OpenVPN Access Server (this lab)** | Individual users connect to AWS VPC, but **you** manage the VPN server (EC2 instance) | Self-managed EC2 instance | OpenVPN Connect app |

The key trade-off: AWS Client VPN is fully managed (less to break, more expensive per connection-hour), while OpenVPN AS is a plain EC2 instance you patch, secure, and pay for at normal EC2 rates — cheaper at small scale, but you own the operational burden.

### 2.3 Why the private EC2 has no public IP

The entire point of the lab is to prove that a resource with **zero internet exposure** can still be reached — but only by someone who has authenticated through the VPN. This is the same "private-by-default, VPN/bastion-by-exception" pattern used in NAT Gateway and SSM Session Manager labs, just with a different access mechanism (VPN tunnel instead of NAT or SSM).

### 2.4 Why `Allow access from all server-side private subnets` matters

This single checkbox, set per-user in OpenVPN AS, controls **split-tunnel routing into your VPC**. Without it, the VPN client only gets a tunnel to the OpenVPN AS instance itself, not a route into `10.0.0.0/24`. This is the setting most people forget, and it is the #1 cause of "VPN connects fine but I can't reach my private EC2."

### 2.5 Why `Route Client Traffic = no`

Setting this to `no` during `ovpn-init` avoids **full-tunnel mode**, where *all* of the client's internet traffic (browsing, everything) would route through the OpenVPN AS instance. For this lab we only want **split-tunnel** access to the VPC CIDR, not to hijack the client's general internet traffic.

---

## 3. Architecture

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/34cd6ffc-5fc2-41d5-a323-be868e3f8b85" />


**Route tables**

| Subnet | Destination | Target |
|---|---|---|
| Public | `10.0.0.0/16` | local |
| Public | `0.0.0.0/0` | Internet Gateway |
| Private | `10.0.0.0/16` | local |

Note there is deliberately **no NAT Gateway route** in the private route table — the private EC2 has no path to the internet at all. It is reachable exclusively through the VPN tunnel, which routes over the VPC's internal `local` route.

---

## 4. CIDR Plan (for this lab, non-overlapping with other labs in this series)

| Resource | CIDR / Range |
|---|---|
| VPC | `10.0.0.0/16` |
| Public Subnet (OpenVPN AS) | `10.0.1.0/24` |
| Private Subnet (workload EC2) | `10.0.0.0/24` |
| VPN Dynamic Pool (client IP pool) | `172.27.224.0/20` |
| VPN Default Group subnet | `172.27.240.0/20` |
| VPN DNS relay range | `100.64.0.0/10` |

> If you're running this alongside the Site-to-Site VPN or VPC Peering labs from this series, confirm none of your other VPC CIDRs overlap with `10.0.0.0/16` before deploying, since overlapping CIDRs will silently break routing if you ever peer or connect these VPCs together.

---

## 5. Prerequisites

- An AWS account with permissions to create VPCs, EC2 instances, and accept AWS Marketplace subscriptions.
- An EC2 Key Pair (create one or reuse an existing `.pem` file).
- OpenVPN Connect client installed locally: [https://openvpn.net/client/](https://openvpn.net/client/) (Windows/Mac/Linux).
- Your local machine's public IP (for locking down SSH access to the OpenVPN AS instance) — check via `curl ifconfig.me`.

---

## 6. Lab Steps

### Step 1 — Create the VPC

- Go to **VPC → Your VPCs → Create VPC**.
- CIDR block: `10.0.0.0/16`
- Enable **DNS resolution**: Yes
- Enable **DNS hostnames**: Yes
- Name: `vpc-openvpn-lab`

### Step 2 — Create Subnets

**Public Subnet**
- CIDR: `10.0.1.0/24`
- Availability Zone: any (note it down, e.g. `ap-south-1a`)
- Enable **Auto-assign public IPv4 address**

**Private Subnet**
- CIDR: `10.0.0.0/24`
- Same or different AZ (same AZ is fine for a lab)
- Disable **Auto-assign public IPv4 address**

### Step 3 — Internet Gateway

- Create an Internet Gateway, name it `igw-openvpn-lab`.
- Attach it to `vpc-openvpn-lab`.

### Step 4 — Route Tables

**Public Route Table** (`rt-public-openvpn`)
- Associate with the public subnet.
- Routes: `10.0.0.0/16 → local`, `0.0.0.0/0 → igw-openvpn-lab`.

**Private Route Table** (`rt-private-openvpn`)
- Associate with the private subnet.
- Routes: `10.0.0.0/16 → local` only. **Do not add a `0.0.0.0/0` route here.**

### Step 5 — Launch the Private EC2 Instance

- AMI: Ubuntu Server 26.04 LTS
- Subnet: private subnet
- Auto-assign public IP: **disabled**
- Key pair: your existing key pair

**Security Group** (`sg-private-workload`)

| Type | Protocol | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | `10.0.0.0/16` |
| ICMP | ICMP | All | `10.0.0.0/16` |

- Launch, then note the **private IPv4 address** (example used throughout this guide: `10.0.0.59`).

### Step 6 — Launch OpenVPN Access Server

- Go to **AWS Marketplace** and search: **OpenVPN Access Server / Self-Hosted VPN (BYOL)**.
- Subscribe, then **Launch through EC2**.
- Instance type: `t2.small` (minimum recommended; `t3.small` also works if available in your region).
- Network: `vpc-openvpn-lab`, **public subnet**.
- Auto-assign public IP: **enabled** (or attach an Elastic IP after launch — recommended for a stable endpoint).
- In Firewall (security groups) Choose Create security group don’t creates new SG Opnevpn have there own SG Groups we Just have to Change SSH Source Only
- Key pair: your existing key pair.
- After launch: select the instance → **Actions → Networking → Change source/destination check → Stop**. This is mandatory — without disabling this check, the instance will silently drop forwarded VPN traffic.

### Step 7 — Security Group for OpenVPN AS

**Inbound rules** (`sg-openvpn-as`)

| Type | Protocol | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | Your public IP `/32` |
| HTTPS (Admin/Client UI) | TCP | 943 | `0.0.0.0/0` |
| HTTPS (VPN over TCP) | TCP | 443 | `0.0.0.0/0` |
| Custom UDP (VPN) | UDP | 1194 | `0.0.0.0/0` |

**Outbound:** Allow all (default).

> Locking SSH (22) to your own `/32` is a real security control, not lab pedantry — port 943 and 443 are meant to be public-facing (that's the VPN's job), but SSH to the management port of your VPN gateway should not be.

### Step 8 — SSH into the OpenVPN AS Instance

```bash
chmod 400 vpnkeys.pem
ssh -i vpnkeys.pem openvpnas@<OPENVPN-AS-PUBLIC-IP>
```

### Step 9 — Run Initial Configuration (`ovpn-init`)

```bash
sudo ovpn-init
```

Answer the prompts exactly as follows:

| Prompt | Answer |
|---|---|
| Will this be the primary Access Server node? | `yes` |
| Network interface | `0.0.0.0` (all interfaces) |
| Use the default CA certificate settings? | Custom → `secp384r1` |
| Web server certificate | `secp384r1` |
| Admin UI port | `943` |
| VPN (TCP) port | `443` |
| Should client traffic be routed by default through the VPN? | `no` |
| Should client DNS traffic be routed by default through the VPN? | `no` |
| Allow access to private subnets? | `yes` |
| Admin username | `openvpn` |
| Set admin password | *(choose a strong password)* |
| Activation key | *(leave blank — free 2-connection license)* |

Once complete, the CLI prints the Admin UI URL:

```text
https://<PUBLIC-IP>:943/admin
```

### Step 10 — Log In to the Admin UI

- Navigate to `https://<PUBLIC-IP>:943/admin` in a browser (accept the self-signed cert warning).
- Username: `openvpn`
- Password: the one set during `ovpn-init`.

### Step 11 — Configure VPN Server Network Settings

Under **Configuration → Network Settings**:

| Setting | Value |
|---|---|
| Hostname or IP Address | Public IP (or the Elastic IP if attached) |
| Interface | All Interfaces |
| Protocol | TCP and UDP |
| TCP Port | 443 |
| UDP Port | 1194 |

Click **Save Settings**, then **Update Running Server** to apply and restart.

### Step 12 — Configure VPN Subnets

Under **Configuration → VPN Settings**, leave the defaults:

| Setting | Value |
|---|---|
| Dynamic IP Address Pool | `172.27.224.0/20` |
| Default Client Group Subnet | `172.27.240.0/20` |
| DNS Relay Range | `100.64.0.0/10` |

Save and update the running server.

### Step 13 — Create a VPN User

Under **User Management → User Permissions → New**:

| Field | Value |
|---|---|
| Username | e.g. `User Name` |
| Password | *(strong password)* |
| User Role | `User` |

Under that user's **Show Options → Networking**:

- IP Address: **Use Dynamic Address**
- ✅ **Allow Access From all Server-side Private Subnets** *(this is the critical checkbox from §2.4)*
- ⬜ Leave **Allow Access From all Other VPN Clients** unchecked (no lateral client-to-client access needed for this lab)

Save, then **Update Running Server**.

### Step 14 — Download and Import the VPN Profile

- In a browser, go to `https://<PUBLIC-IP>/` (client portal). No port number is needed here — OpenVPN AS shares port 443 between the raw OpenVPN protocol and a web redirect into the Client UI, so plain HTTPS on the root domain lands you on the login page.
- Log in with the VPN user you just created (e.g. `saime`).
- You'll see two profile options — pick deliberately, since they behave differently:
  - **Yourself (autologin profile)** — embeds the user's certificate/key; OpenVPN Connect connects with no password prompt. Convenient for a lab, less safe if the file is lost/stolen.
  - **Yourself (user-locked profile)** — requires the username/password every time you connect. Better for real deployments.
- Download whichever profile (`client.ovpn`) suits your case, and install the **OpenVPN Connect** desktop/mobile app if you haven't already.
- In OpenVPN Connect: **Import Profile → File → select `client.ovpn`** → (enter credentials if you downloaded the user-locked profile) → **Connect**.

### Step 15 — Verify the VPN Tunnel

- OpenVPN Connect should show **Connected**, with an assigned IP from `172.27.224.0/20`.
- On some clients, click the connection details to confirm the assigned tunnel IP and uptime.

### Step 16 — SSH into the Private EC2 Over the Tunnel

```bash
ping 10.0.0.59
ssh -i vpnkey.pem ubuntu@<10.0.0.59/add your priavte IP hare >
```

<img width="3060" height="1040" alt="image" src="https://github.com/user-attachments/assets/f7411ec9-eee9-48e9-9ca4-bb06c9d71b5e" />


If this succeeds, you have proven full-path connectivity: laptop → VPN tunnel → OpenVPN AS → VPC routing → private subnet → EC2, with the private instance never having touched the public internet.

---


---

## 9. Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Can't reach `https://<PUBLIC-IP>:943/admin` | SG missing inbound 943, or instance still initializing | Confirm SG rule for 943/tcp from `0.0.0.0/0`; wait 2–3 min after launch |
| VPN client connects but private EC2 unreachable | User missing "Allow access from all server-side private subnets" | Edit user in Admin UI → Networking → enable the checkbox → Update Running Server |
| VPN client can't connect at all (timeout) | SG missing 443/tcp or 1194/udp, or Source/Dest check still enabled | Verify SG rules; verify Source/Dest check is disabled on the OpenVPN AS instance |
| `ovpn-init` fails or hangs | Wrong instance type (too small) or AMI not the official Marketplace BYOL image | Re-launch on `t2.small`+ using the exact Marketplace listing |
| SSH to private EC2: `Permission denied (publickey)` | Wrong `.pem` file used vs. the key pair attached to that specific instance | Match the key pair name shown in the EC2 console to the correct `.pem` |
| SSH to private EC2: `Operation timed out` | VPN not actually connected, SG rule missing, or wrong route table | Confirm VPN shows Connected; confirm `sg-private-workload` allows 22 from `10.0.0.0/16`; confirm private RT has `10.0.0.0/16 → local` |
| Ping works but SSH fails | ICMP allowed, TCP 22 blocked, or `sshd` not running | Check SG allows TCP 22 from VPC CIDR; check instance system log |
| OpenVPN Connect never prompts for a password / connects instantly | You downloaded the **autologin** profile, which embeds credentials by design | This is expected behavior, not a bug — download the **user-locked** profile instead if you want a login prompt each time |
| OpenVPN Connect keeps asking for a password you don't have | You downloaded the **user-locked** profile but don't have the VPN user's password | Re-download the autologin profile, or get the correct password from whoever created the user |
| Full internet traffic gets routed through VPN unexpectedly | `Route Client Traffic` was set to `yes` during `ovpn-init` | Re-run `ovpn-init` or change in Admin UI → VPN Settings → Routing, set to split-tunnel |
| Only 2 users can connect simultaneously | Free OpenVPN AS license limit | Purchase additional connection licenses, or use fewer concurrent test users |

---

## 10. Interview Q&A

**Q1: Why does the private EC2 instance have no public IP, and how does it still get internet-independent remote access?**
A: It relies entirely on VPC-internal routing. The VPN tunnel terminates at the OpenVPN AS instance in the public subnet; authorized client traffic is then routed over the VPC's `local` route into the private subnet. The private instance never needs, and never gets, a route to `0.0.0.0/0`.

**Q2: What's the difference between OpenVPN Access Server and AWS Client VPN?**
A: AWS Client VPN is a fully managed AWS service — no EC2 instance to patch, billed per connection-hour and per subnet association, integrates natively with ACM for certificates. OpenVPN AS is a self-managed EC2 instance running commercial OpenVPN software — you own patching, scaling, and PKI management, but you pay standard EC2 rates and have full control over the box.

**Q3: Why must "Source/Destination Check" be disabled on the OpenVPN AS instance?**
A: By default, AWS drops any packet on an ENI whose source or destination IP doesn't match the instance's own IP, to prevent accidental traffic forwarding. Since OpenVPN AS's entire job is to forward VPN client traffic to the private subnet (traffic *not* originating from or destined to the instance itself), this check must be disabled — exactly the same requirement as for NAT instances.

**Q4: What does "split-tunnel" mean and why is it used here?**
A: Split-tunnel routes only traffic destined for specific CIDRs (here, `10.0.0.0/16`) through the VPN, while all other client internet traffic goes out the client's normal internet connection. It's set via `Route Client Traffic = no` during `ovpn-init`. The alternative, full-tunnel, routes 100% of client traffic through the VPN — useful for content filtering or full network isolation, but unnecessary overhead for pure VPC access.

**Q5: If SSH from the VPN client to the private EC2 times out, what's your troubleshooting order?**
A: 1) Confirm VPN client shows "Connected" with an assigned tunnel IP. 2) Confirm the VPN user has "Allow access from all server-side private subnets" enabled. 3) Confirm the private EC2's security group allows port 22 from the VPC CIDR. 4) Confirm the private route table only has the `local` route (no conflicting routes). 5) Confirm Source/Dest check is disabled on the OpenVPN AS instance.

**Q6: How would you scale this pattern for a 50-person engineering team?**
A: The free OpenVPN AS license only supports 2 concurrent connections — you'd need to purchase additional connection licenses (or move to AWS Client VPN, which scales more natively). You'd also want to move from a single `t2.small` to a larger instance type or an Auto Scaling setup behind a Network Load Balancer for HA, plus integrate with an external IdP (LDAP/SAML) instead of local OpenVPN AS user accounts.

**Q7: What security risk exists if you leave SSH (22) open to `0.0.0.0/0` on the OpenVPN AS security group?**
A: The OpenVPN AS instance is your single point of entry into the entire private network — compromising its SSH access is equivalent to compromising the whole VPC's private workloads. Restricting SSH to a known admin IP (or better, using SSM Session Manager instead of SSH entirely) significantly reduces that attack surface.

---

## 11. Cheat Sheet

```bash
# SSH to OpenVPN AS
ssh -i vpnkeys.pem openvpnas@<OPENVPN-AS-PUBLIC-IP>

# Re-run initial config if needed
sudo ovpn-init

# Admin UI
https://<PUBLIC-IP>:943/admin

# Client portal (download .ovpn profile here, NOT from :943)
https://<PUBLIC-IP>/

# Connect via CLI (Linux, alternative to GUI client)
sudo openvpn --config client.ovpn

# Verify tunnel + reach private EC2
ping 10.0.0.59
ssh -i vpnkey.pem ubuntu@10.0.0.59

# Check OpenVPN AS service status (on the AS instance)
sudo systemctl status openvpnas

# View OpenVPN AS logs
sudo tail -f /var/log/openvpnas.log
```

**Critical checklist items people forget:**
- ✅ Disable Source/Destination Check on the OpenVPN AS instance.
- ✅ Enable "Allow access from all server-side private subnets" per user.
- ✅ Download the connection profile from the **client** portal (port 443/root), not the admin portal (943).
- ✅ Private subnet route table has **no** `0.0.0.0/0` route.

---

## 12. Mastery Checklist

- [ ] I can explain why the private EC2 has no public IP and no NAT route, yet is reachable.
- [ ] I can explain the difference between OpenVPN AS, AWS Client VPN, and Site-to-Site VPN.
- [ ] I disabled Source/Destination Check and can explain why it's required.
- [ ] I configured split-tunnel routing via `Route Client Traffic = no`.
- [ ] I enabled per-user access to server-side private subnets and can explain what it controls.
- [ ] I downloaded the `.ovpn` profile from the correct (client) portal.
- [ ] I successfully pinged and SSH'd into the private EC2 through the tunnel.
- [ ] I can diagnose "VPN connects but private resource unreachable" without looking anything up.
- [ ] I can rebuild the network layer (VPC/subnets/routes/SGs) using the AWS CLI reference.
- [ ] I can deploy the workload + networking layer via Terraform (excluding the Marketplace subscription step).

---

