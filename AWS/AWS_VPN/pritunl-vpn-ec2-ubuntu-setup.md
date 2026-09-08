# Setting Up Pritunl VPN on EC2 Ubuntu — End-to-End Guide

Pritunl is a self-hosted, open-source VPN server with a web-based management console — it's the practical, real-world version of the **Point-to-Site VPN** concept: individual devices (your laptop, phone) connect into it, and it hands each device a private IP so it can reach whatever network you've configured behind it. This guide installs it on a single Ubuntu EC2 instance, end to end, with every command run separately and explained.

---

## Part 1 — Understand the architecture first

<img width="1167" height="1347" alt="image" src="https://github.com/user-attachments/assets/1c55735c-3670-4bfa-91dc-ac45977505a1" />


**What each piece is doing, in plain terms:**

| Piece | Role |
|---|---|
| **The EC2 instance** | Runs both Pritunl and its database — everything lives on one machine for this setup. |
| **MongoDB** | Pritunl's own database — it stores every organization, user, VPN server config, and certificate you create through the web console. Pritunl is unusable without it. |
| **Pritunl process** | Two things at once: the **web console** (the admin dashboard you configure everything through) and the **VPN server process** itself (the thing that actually terminates client tunnels). |
| **Elastic IP** | The instance's public IP must stay fixed — client `.ovpn` profiles have this IP baked into them, so if it changed, every existing client profile would break. |
| **Organization** | A grouping/namespace for users inside Pritunl — think of it like a team or department. Every user belongs to one. |
| **User** | An individual person who gets their own certificate and `.ovpn` profile — this is the actual "point" in Point-to-Site. |
| **Server** | The actual VPN listener Pritunl runs (protocol, port, IP range handed out to clients). An organization is attached to a server to allow its users to connect through it. |
| **Client** | The Pritunl client app (or any OpenVPN-compatible client) running on your laptop/phone, using the downloaded profile to connect. |

---

## Part 2 — Prerequisites

1. **An AWS account** with permission to launch EC2 instances, allocate Elastic IPs, and edit security groups.
2. **Ubuntu 24.04 LTS as the AMI.** This guide uses the officially documented Ubuntu 24.04 install path from Pritunl's own docs — other Ubuntu versions aren't guaranteed to work the same way.
3. **One important thing to know before you start:** Pritunl's own documentation is explicit that **RHEL-based distributions (Oracle Linux, AlmaLinux, Amazon Linux) get continuous testing and dedicated builds, while Ubuntu support is provided "as-is" and not guaranteed for future releases.** This guide still uses Ubuntu, per your request — just know that if you hit a wall later (an OS upgrade breaking something), that's a known trade-off of choosing Ubuntu over Amazon Linux/Oracle Linux for this specific piece of software.
4. **An instance type with at least 1 vCPU / 1GB RAM** — `t3.small` or larger is a comfortable choice; `t2.micro` can work for a personal test but is tight once MongoDB is running alongside Pritunl.
5. **A key pair** to SSH into the instance.
6. **Basic comfort with the Linux command line** — you'll be adding package repositories and editing one config value.
7. **A way to view the AWS-assigned public IP** before you allocate the Elastic IP, so you can reach the web console for the very first login.

---

## Part 3 — Hands-on, step by step

### Step 1 — Launch the EC2 instance

1. **EC2 → Launch instance.**
2. Name: `pritunl-vpn`.
3. AMI: **Ubuntu Server 24.04 LTS**.
4. Instance type: `t3.small` (or larger).
5. Key pair: select yours.
6. Leave networking on your default VPC/subnet — this doesn't need to be a private subnet; the whole point is that it needs to be reachable from the internet.
7. Launch.

*Why Ubuntu 24.04 specifically:* Pritunl's officially documented Ubuntu install path targets 24.04 (codename `noble`) — using an older or newer Ubuntu release means guessing at repository codenames that may not exist, which is exactly the kind of thing that causes silent package-not-found failures later.

### Step 2 — Allocate and attach an Elastic IP

1. **EC2 → Elastic IPs → Allocate Elastic IP address.**
2. Select it → **Actions → Associate Elastic IP address** → choose `pritunl-vpn`.

*Why now, before installing anything:* client VPN profiles you download later will have this exact public IP embedded in them. If you install first and attach the Elastic IP afterward, that's fine too — just make sure the IP is attached and stable **before** you download any client profiles, since a profile downloaded against the wrong IP simply won't connect.

### Step 3 — Configure the security group

Edit the security group attached to `pritunl-vpn` and add these inbound rules:

| Type | Port | Source | Why |
|---|---|---|---|
| SSH | 22 | Your IP | Management access only — never open this to `0.0.0.0/0`. |
| Custom TCP | 443 | Your IP (or `0.0.0.0/0` if you'll manage it from multiple locations) | The web console — this is where you'll log in to configure everything. |
| Custom TCP | 80 | `0.0.0.0/0` | Used for HTTP→HTTPS redirect and Let's Encrypt certificate verification if you set up a real SSL certificate later. |
| Custom UDP | 1194 | `0.0.0.0/0` | The default OpenVPN tunnel port — this is what actual VPN clients connect to. Change this to whatever port you pick inside Pritunl if you choose a different one. |

*Why 1194/UDP needs to stay open to everyone, unlike SSH:* your VPN clients could be connecting from anywhere — home, a coffee shop, another country — so unlike SSH (which you access from known locations), the VPN port has to accept connections from any public IP by design. The security here comes from the certificate-based authentication inside the tunnel, not from restricting who can reach the port.

### Step 4 — SSH into the instance and prepare it

```bash
ssh -i your-key.pem ubuntu@<your-elastic-ip>
```
Install `gnupg`, which is needed to import the repository signing keys in the next steps:
```bash
sudo apt --assume-yes install gnupg
```

### Step 5 — Add the MongoDB repository

Create the repository definition file:
```bash
sudo tee /etc/apt/sources.list.d/mongodb-org.list << EOF
deb [ signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse
EOF
```
Import MongoDB's signing key into its own dedicated keyring file:
```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor --yes
```

*Why a dedicated keyring file per repository (`signed-by=...`), instead of one shared system keyring:* this is the modern, more secure `apt` practice — each repository's package signatures are checked only against its own specific key file, rather than trusting any key ever added system-wide. It also avoids the deprecated `apt-key` command entirely.

### Step 6 — Add the OpenVPN repository

Pritunl's official docs specifically add a dedicated OpenVPN build, rather than relying on Ubuntu's own (older) OpenVPN package.

Create the repository definition file:
```bash
sudo tee /etc/apt/sources.list.d/openvpn.list << EOF
deb [ signed-by=/usr/share/keyrings/openvpn-repo.gpg ] https://build.openvpn.net/debian/openvpn/stable noble main
EOF
```
Import its signing key:
```bash
curl -fsSL https://swupdate.openvpn.net/repos/repo-public.gpg | sudo gpg -o /usr/share/keyrings/openvpn-repo.gpg --dearmor --yes
```

*Why this matters:* this is the exact issue behind Pritunl's official warning about older Ubuntu OpenVPN builds — using the distro's default `openvpn` package instead of this dedicated repository is a common cause of client authentication failures on Ubuntu installs.

### Step 7 — Add the Pritunl repository

Create the repository definition file:
```bash
sudo tee /etc/apt/sources.list.d/pritunl.list << EOF
deb [ signed-by=/usr/share/keyrings/pritunl.gpg ] https://repo.pritunl.com/stable/apt noble main
EOF
```
Import its signing key:
```bash
curl -fsSL https://raw.githubusercontent.com/pritunl/pgp/master/pritunl_repo_pub.asc | sudo gpg -o /usr/share/keyrings/pritunl.gpg --dearmor --yes
```

### Step 8 — Refresh package lists and install everything

```bash
sudo apt update
```
Install Pritunl, OpenVPN, MongoDB, and WireGuard support together:
```bash
sudo apt --assume-yes install pritunl openvpn mongodb-org wireguard wireguard-tools
```

*Why WireGuard is installed even though this guide uses OpenVPN:* Pritunl supports both protocols from the same server, and installing WireGuard support now means you can enable it later from the web console with no further package changes.

### Step 9 — Disable the local firewall (ufw)

```bash
sudo ufw disable
```

*Why disable it rather than just opening specific ports in it:* Pritunl manages its own `iptables` rules internally to route VPN client traffic correctly. If `ufw` is also active and managing `iptables` at the same time, the two can conflict — Pritunl's official install instructions disable `ufw` specifically to avoid this. This is safe here because your **security group is the actual firewall** protecting this instance at the network level; `ufw` would only have been a second, redundant layer, and in this case an actively conflicting one.

### Step 10 — Start and enable the services

Start MongoDB:
```bash
sudo systemctl start mongod
```
Enable it to start on every boot:
```bash
sudo systemctl enable mongod
```
Start Pritunl:
```bash
sudo systemctl start pritunl
```
Enable it to start on every boot:
```bash
sudo systemctl enable pritunl
```

### Step 11 — Disable source/destination check (only if clients will reach a private network through this server)

If you only want your devices to reach the *internet* through this VPN (or just reach the Pritunl box itself), skip this step. If you want VPN clients to reach **other private resources** in your VPC (other EC2 instances, an RDS database, etc.), this instance needs to act as a router:

1. **EC2 → select `pritunl-vpn` → Actions → Networking → Change source/destination check → Disable.**

*Why, precisely:* this check only matters when a packet leaving this instance's network card has a **source IP that isn't the instance's own IP** — which happens when Pritunl forwards a VPN client's traffic to the VPC **without NAT**, since the client's original IP is preserved as-is. If you enable NAT on your routes (the default recommendation in Step 18), outbound packets already carry this instance's own IP as the source by the time they reach the VPC, so the check wouldn't actually block them either way. **Disabling it now is still the right call regardless** — it costs nothing, and it's what makes the non-NAT path in Part 4 possible later without having to come back and change it.

### Step 12 — Retrieve the setup key

```bash
sudo pritunl setup-key
```

This prints a setup key to the terminal — copy it, you'll need it in the next step.

### Step 13 — Open the web console and complete database setup

1. In a browser, go to `https://<your-elastic-ip>`. Accept the self-signed certificate warning (expected — you haven't configured a real SSL certificate yet).
2. Paste in the **setup key** from Step 12.
3. Leave the **MongoDB URI** field at its default value — it already points to the MongoDB instance running on this same server.
4. Save.

### Step 14 — Log in and complete initial setup

1. Default username: **`pritunl`**.
2. Get the default password by running this on the server:
   ```bash
   sudo pritunl default-password
   ```
3. Log in with `pritunl` and that password.
4. The initial setup dialog appears — change the username and password to something of your own here. The server's public address is auto-detected and normally doesn't need changing.

*Why change the password immediately:* the default password is generated but still predictable-format enough that leaving it unchanged on an internet-facing admin console is a real risk — do this before creating any organizations or users.

### Step 15 — Create an organization

1. In the web console, go to **Users → Organizations → Add Organization**.
2. Name it (e.g. `default-org`).

*Why organizations exist at all:* they're how Pritunl groups users for access control — every server you create gets attached to one or more organizations, and only users in an attached organization can connect through that server. For a single-person lab, one organization is enough.

### Step 16 — Create a user

1. **Users → Add User.**
2. Organization: the one you just created. Name: your own name or a device label.
3. Save.

*What happens behind the scenes:* Pritunl generates a unique certificate for this user — this certificate, not a password, is what actually authenticates the VPN connection later.

### Step 17 — Create a server

1. **Servers → Add Server.**
2. Name it (e.g. `main-server`).
3. Protocol: **UDP**. Port: Pritunl fills this in with a **random** UDP port by default — change it to **1194** so it matches the security group rule from Step 3 (or pick any port you like, as long as you update the security group to match exactly).
4. Network: Pritunl also fills this in with a **random** private CIDR by default — change it to something predictable like `192.168.0.0/24`, which this guide uses from here on. It just needs to not overlap with your own local network at home (check your router's IP range if unsure) and be large enough for however many users you'll attach.
5. Save.

*Why call out that these are randomized, not fixed defaults:* if you don't explicitly set them, you'll get a server with some arbitrary port that doesn't match the UDP rule you already opened in Step 3, and a VPN network CIDR you didn't choose — both are easy to overlook since the form appears pre-filled and looks "done" already.

### Step 18 — Configure server routes (the step that actually determines what clients can reach)

This is separate from the internal VPN network you set in Step 17, and easy to miss — a server can be attached, started, and show a client as "connected" while this step is still wrong, because the tunnel itself works fine even when routing is misconfigured.

By default, every new server includes one route: **`0.0.0.0/0`**, meaning "send *all* of this client's internet traffic through the VPN." What you do next depends on what you actually want:

**Option A — Full-tunnel (all client internet traffic goes through this server):**
1. Go to the server → **Routes** tab.
2. Leave the default `0.0.0.0/0` route in place.
3. Make sure **NAT Route** is enabled on it.

*Why NAT has to be on:* with `0.0.0.0/0` active, every connected client's internet-bound packets arrive at this EC2 instance expecting it to forward them onward to the real internet. Without NAT, this instance would try to route those packets natively and normally fail — NAT rewrites them to look like they originated from the instance itself, exactly what your home router does for every device on your Wi-Fi. This is the single most common reason a "connected" VPN client suddenly has no internet at all.

**Option B — Split-tunnel (client only reaches a specific private network, e.g. your VPC, general internet browsing stays on the client's own connection):**
1. Go to the server → **Routes** tab.
2. **Remove** the default `0.0.0.0/0` route.
3. Click **Add Route** and enter the specific CIDR you want reachable — e.g. your VPC's CIDR (`10.0.0.0/16`), or any other private network this instance can reach.
4. **Enable NAT Route** on this one too, unless you've separately added a route in the VPC's own route table pointing that CIDR back at this Pritunl instance (covered in Part 4 below) — Pritunl's own documentation states NAT is required "unless a static route is configured on the router for the vpn network," and by default, your VPC's route table has no idea the VPN client network exists.

*Why you'd choose split-tunnel:* it's the more realistic Point-to-Site setup for reaching a specific internal network — the client's general web browsing keeps using its own internet connection directly, and only traffic genuinely destined for your private CIDR goes through the tunnel. This is both faster (no unnecessary detour for ordinary browsing) and closer to what "VPN into the office network" usually means in practice.

*A note on DNS:* Pritunl sets the DNS server clients use to Google's public DNS by default — this is fine for Option A, but if you're doing Option B specifically to reach internal resources by hostname (not just IP), you'll need to point this at your own internal DNS server instead, in the same server configuration screen.

### Step 19 — Attach the organization to the server, and start it

1. On the server you configured, click **Attach Organization** → select your organization.
2. Click **Start Server**.

*Why attaching is a separate step from creating:* this separation is what lets one Pritunl installation serve multiple organizations with different access levels through different servers — for this lab it's a single extra click, but it's the mechanism that scales later.

### Step 20 — Download the user's profile

1. Go back to **Users**, find the user you created, and click the **download** icon next to their name.
2. This downloads a `.ovpn` profile file — this is what the client device needs to connect.

### Step 21 — Install the Pritunl client and connect

1. On your laptop, download the **Pritunl Client** from Pritunl's official client download page for your OS (Windows, macOS, or Linux).
2. Install it, open it, and **import** the `.ovpn` profile downloaded in Step 20.
3. Click **Connect**.

*What happens when you connect:* your laptop authenticates using the certificate embedded in that profile, Pritunl assigns it an IP from the internal range you set in Step 17, and — depending on which routing option you chose in Step 18 — either all of its internet traffic or just traffic to your specified private CIDR now goes through this encrypted tunnel.

### Step 22 — Verify the connection

On your laptop, once connected:
```bash
ip addr show tun0
```
You should see an IP address from the internal range configured in Step 17 (e.g. `192.168.0.x`) assigned to a `tun0` interface — that confirms the tunnel is genuinely up and you've been handed a VPN IP.

If you chose **Option A (full-tunnel)**, also confirm actual internet access works through the tunnel — visit any site and confirm it loads. If you chose **Option B (split-tunnel)**, try reaching something specifically inside the CIDR you routed — e.g. pinging another EC2 instance's private IP in that VPC. Reaching Step 21 with a `tun0` interface assigned is necessary but **not sufficient** on its own — the Step 18 routing choice is what determines whether either of these actually succeeds.

---

## Part 4 — AWS VPC Routing — End-to-End Packet Flow

This is the part that determines whether Option B (split-tunnel to your VPC) actually works, and it lives partly outside Pritunl's own web console — in AWS's VPC route tables and security groups. There are two genuinely different paths, and they need different AWS-side configuration.

### Path 1 — With NAT Route enabled (the default recommendation from Step 18)

```
VPN client                Pritunl EC2 (pritunl-vpn)              Target EC2 (in your VPC)
192.168.0.5                 eth0: 10.0.0.5 (private)                10.0.0.20 (private)
    │                                                                     │
    │  packet: src=192.168.0.5, dst=10.0.0.20                            │
    ├──── via tun0 (VPN tunnel) ─────────────▶                           │
    │                                    NAT rewrites source:            │
    │                                    src=10.0.0.5, dst=10.0.0.20     │
    │                                    ├──── via eth0, normal VPC ────▶│
    │                                    │      routing (local route)    │
    │                                    │                                │
    │                                    │◀──── reply: src=10.0.0.20, ───┤
    │                                    │      dst=10.0.0.5              │
    │                                    NAT reverses it back to:         │
    │                                    src=10.0.0.20, dst=192.168.0.5  │
    │◀──── via tun0 ──────────────────────                                │
```

**What this means for AWS configuration:**
- **VPC route table: no changes needed at all.** As far as the VPC is concerned, the traffic reaching the target instance came from `pritunl-vpn`'s own private IP (`10.0.0.5`) — an address the VPC's automatic "local" route already knows how to deliver to and receive replies from. The VPN client CIDR (`192.168.0.0/24`) never appears inside the VPC's own routing at all.
- **Security group on the target instance:** must allow inbound traffic **from the Pritunl instance's private IP or its security group** — not from the VPN client CIDR, since by the time the packet arrives, NAT has already rewritten the source to be the Pritunl instance's own address.
- **This is why Path 1 is the simpler, recommended default** — it requires zero VPC route table edits, at the cost of the target instance seeing all VPN client traffic as if it came from one shared address (`pritunl-vpn`'s IP), rather than each client's own distinct VPN IP.

### Path 2 — Without NAT (preserving each client's real VPN IP)

```
VPN client                Pritunl EC2 (pritunl-vpn)              Target EC2 (in your VPC)
192.168.0.5                 eth0: 10.0.0.5 (private)                10.0.0.20 (private)
    │                                                                     │
    │  packet: src=192.168.0.5, dst=10.0.0.20                            │
    ├──── via tun0 ───────────────────────────▶                          │
    │                                    forwarded AS-IS, no rewrite:     │
    │                                    src=192.168.0.5, dst=10.0.0.20  │
    │                                    ├──── via eth0 ─────────────────▶│
    │                                    │                                │
    │                                    │◀──── reply: src=10.0.0.20, ───┤
    │                                    │      dst=192.168.0.5           │
    │                                    │      (needs a route!)          │
    │◀──── via tun0 ───────────────────────                               │
```

**What this means for AWS configuration — this is the part that was missing:**
- **VPC route table: a manual entry is required.** The target instance's reply is addressed to `192.168.0.5` — a CIDR the VPC has never heard of and has no local route for. Without an explicit route, this reply simply has nowhere to go and silently fails. You must add a route to the route table associated with the target instance's subnet: destination `192.168.0.0/24` (your VPN client network from Step 17), target = the `pritunl-vpn` instance (selected as an **Instance** target, the same way you'd point traffic at a NAT instance).
- **Security group on the target instance:** must allow inbound traffic **from the actual VPN client CIDR** (`192.168.0.0/24`) directly, since the client's real IP is preserved end to end.
- **Why you'd choose this harder path anyway:** it lets you write security group rules and VPC Flow Logs that distinguish between individual VPN users by their actual assigned IP, rather than seeing all of them collapse into one shared address. Pritunl also offers an automated version of this (**AWS Route Advertisement**, using an IAM role with VPC permissions to add/update this route automatically, including on failover) — worth knowing exists, but out of scope for a single-server lab.

### Verifying the packet flow actually works

Don't just trust that the client shows "connected" — confirm the path end to end:

1. **On the target EC2 instance**, watch incoming traffic while you ping it from the connected VPN client:
   ```bash
   sudo tcpdump -i eth0 icmp
   ```
2. Compare the **source IP you see here** against which path you configured:
   - Path 1 (NAT): you should see the source as `pritunl-vpn`'s private IP (`10.0.0.5` in this example) — **not** the VPN client's `192.168.x.x` address.
   - Path 2 (no NAT): you should see the source as the VPN client's actual `192.168.x.x` address.
3. If you see **no packets arriving at all**, the problem is upstream — check the target instance's security group first, then the VPC route table (Path 2 only), then confirm Step 11's source/destination check is actually disabled on `pritunl-vpn`.
4. If packets arrive but the client-side ping still times out, the reply isn't finding its way back — for Path 2, this almost always means the route table entry described above is missing or points at the wrong instance.

---

## Part 5 — Troubleshooting

| Symptom | Likely cause |
|---|---|
| Can't reach `https://<elastic-ip>` at all | Security group is missing the port 443 rule, or the Elastic IP isn't actually associated with the instance. |
| Web console loads but setup key is rejected | You copied it with extra whitespace, or ran `sudo pritunl setup-key` again (each run can generate a fresh key) — re-run and re-copy carefully. |
| Client can't connect (times out) | Security group is missing the UDP 1194 rule, or the port in the security group doesn't match the port configured on the server in Step 17. |
| Client connects but can't reach other private resources | Source/destination check wasn't disabled (Step 11), or the route for that CIDR is missing in Step 18. |
| Client connects but has no internet at all | The `0.0.0.0/0` route in Step 18 is present but **NAT Route** isn't enabled on it — full-tunnel mode needs NAT to forward client traffic onward. |
| `mongod` fails to start | Check `sudo systemctl status mongod` — a common cause is insufficient RAM on a `t2.micro`; move to `t3.small` or larger. |
| OpenVPN authentication errors on newer clients | This is the exact issue Pritunl's own docs flag — confirm the dedicated OpenVPN repository from Step 6 was actually used, not Ubuntu's default `openvpn` package. |

---

## Part 6 — Cleanup

1. **Web console → Servers → stop and delete the server** (optional — not required for AWS-side cleanup, but tidy).
2. **Terminate the EC2 instance** (`pritunl-vpn`).
3. **Release the Elastic IP** — it bills hourly once unattached, and even while attached to a terminated instance's freed address it no longer serves any purpose.
4. Remove the security group if it isn't used elsewhere.

Releasing the Elastic IP is the step most likely to be forgotten and quietly keep billing after you're done.
