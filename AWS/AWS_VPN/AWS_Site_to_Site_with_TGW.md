# AWS Site-to-Site VPN — End-to-End Hands-On Lab

Simulating a real Site-to-Site VPN entirely inside AWS: one VPC plays the role of your **AWS-side production network**, a second VPC (in a different region, so it's genuinely "remote") plays the role of your **on-premises / corporate network**, with an EC2 instance running IPsec software standing in for your physical office router.

---

## Part 1 — Understand the architecture first

Before touching the console, know exactly what you're building and why each piece exists.

<img width="1831" height="859" alt="image" src="https://github.com/user-attachments/assets/8b9b8ff2-9fd5-464f-8129-47090208e00d" />


**What each piece is doing, in plain terms:**

| Piece | Real-world role | Why it's there |
|---|---|---|
| **VPC-A (Mumbai)** | Your actual AWS production network | Hosts the workload (EC2-A) you want to protect and keep private. |
| **EC2-A** | The server on-prem needs to reach | Has *only* a private IP — never exposed to the internet. This is the whole point of the exercise: prove it's reachable privately. |
| **VGW (Virtual Private Gateway)** | AWS's side of the VPN tunnel | Terminates the IPsec tunnels coming from the "on-prem" side and injects routes into VPC-A. |
| **VPC-B (N. Virginia)** | Stand-in for your physical office / data center | In real life this would be your actual office network, not another VPC — we're using a second region purely so it behaves like a genuinely separate, remote network. |
| **EC2-B** | Stand-in for your on-prem VPN router/firewall | Runs IPsec software (Libreswan, the modern successor to Openswan) and does the same job a Cisco ASA or pfSense box would do at a real office: terminate the tunnel and forward traffic between "internal" and "internet." |
| **Elastic IP on EC2-B** | Your office's static public IP | AWS needs a **fixed** public IP to configure as the Customer Gateway — a regular EC2 public IP changes on stop/start, so it must be an Elastic IP. |
| **The two IPsec tunnels** | The actual encrypted VPN | AWS always creates two, terminating in two different Availability Zones, for redundancy. |

**Why non-overlapping CIDRs matter:** VPC-A is `10.100.0.0/16`, VPC-B is `10.200.0.0/16`. If these overlapped, routing would be ambiguous — a packet destined for `10.100.0.5` could never be told apart from one destined for `10.200.0.5` on the wrong side. Always pick non-overlapping ranges before you start.

**One thing this diagram simplifies:** in a real deployment, VPC-B wouldn't exist at all — your actual office router would sit at the edge of your real office LAN. We're only using a second VPC because we don't have physical hardware to demonstrate with. Everything on the AWS-side (VPC-A, VGW, VPN Connection, Customer Gateway) is *exactly* what you'd configure in a real deployment — nothing about that half is a simulation.

---

## Part 2 — Prerequisites (before you touch the console)

Get every one of these sorted first — most "the tunnel won't come up" frustration traces back to skipping one of these.

1. **An AWS account** with permission to create VPCs, EC2 instances, Elastic IPs, VPN Gateways, Customer Gateways, and Site-to-Site VPN connections in two regions.
2. **Two AWS regions** you're allowed to launch resources in — this guide uses Mumbai (`ap-south-1`) and N. Virginia (`us-east-1`), but any two regions work.
3. **Two non-overlapping CIDR blocks** decided in advance (this guide: `10.100.0.0/16` and `10.200.0.0/16`). Write them down — you'll type them repeatedly.
4. **A key pair** in each region (or one key pair imported into both) so you can SSH into both EC2 instances.
5. **Basic comfort with Linux command line** — you'll be editing config files and running `yum install`, `systemctl`, and checking `ip route` on EC2-B.
6. **Know that Openswan is deprecated.** The diagram/video you're following says "Openswan," but Openswan is no longer maintained and isn't available as a package on current Amazon Linux. Its direct successor is **Libreswan**, which speaks the identical IPsec/IKE protocols and is what AWS's own downloadable configuration templates target today. This guide uses Libreswan — treat it as a drop-in modern replacement for "Openswan" in the diagram.
7. **Use Amazon Linux 2 for EC2-B specifically, not Amazon Linux 2023.** This matters: Libreswan installs with a single `yum install` on Amazon Linux 2, but Amazon Linux 2023's default repositories don't include it at all — you'd have to manually add the Fedora repository first. To keep the lab focused on the VPN itself rather than package-repo troubleshooting, EC2-A can be Amazon Linux 2023 (it needs no special packages), but **EC2-B should be Amazon Linux 2**.
8. **Understand what "static routing" means here**, because that's the mode this lab uses: instead of the two sides automatically exchanging routes via BGP, you manually tell AWS "the network on the other end of this tunnel is `10.200.0.0/16`," and manually tell EC2-B "the network on the other end is `10.100.0.0/16`." BGP is possible but needs a routing daemon and an ASN on the Libreswan side — skip it for this first end-to-end pass.
9. **A way to check your own values as you go** — keep a scratch note of: VPC-A CIDR, VPC-B CIDR, EC2-B's Elastic IP, and the pre-shared key AWS generates. You'll need all four multiple times.

---

## Part 3 — Hands-on, step by step

### Step 1 — Create VPC-A (the AWS production side, Mumbai)

1. Switch the console region to **Mumbai (ap-south-1)**.
2. Go to **VPC → Your VPCs → Create VPC**.
3. Choose **VPC only**. Name: `VPC-A`. IPv4 CIDR: `10.100.0.0/16`. Leave everything else default. Create.

*Why "VPC only" and not the wizard:* the wizard auto-creates subnets and NAT gateways you don't need for this lab — building it manually keeps you in control of exactly what exists, which matters when you're trying to understand the architecture rather than just get it working.

### Step 2 — Create a private subnet in VPC-A

1. **VPC → Subnets → Create subnet**. VPC: `VPC-A`.
2. Name: `VPC-A-private`. Availability Zone: pick one (e.g. `ap-south-1a`). IPv4 CIDR: `10.100.0.0/24`.
3. Create. **Do not** enable auto-assign public IPv4 — this subnet must stay private; that's the entire point.

### Step 3 — Launch EC2-A in the private subnet

1. **EC2 → Launch instance.** Name: `EC2-A`.
2. Amazon Linux 2023, `t2.micro` (or your account's free-tier-eligible type).
3. Key pair: select yours.
4. Network settings → VPC: `VPC-A`, Subnet: `VPC-A-private`, **Auto-assign public IP: Disable**.
5. Create a new security group `EC2-A-SG` with:
   - Inbound: **All ICMP – IPv4**, source `10.200.0.0/16` (so EC2-B can ping it)
   - Inbound: **SSH (22)**, source `10.200.0.0/16` (so you can SSH to it *through* the tunnel later, to confirm the private path actually works)
6. Launch.

*Note:* this instance has no internet access at all — that's intentional and fine. You're not installing anything on it; it's purely the "prove I can reach this privately" target.

### Step 4 — Create VPC-B (the simulated on-prem side, N. Virginia)

1. Switch region to **N. Virginia (us-east-1)**.
2. **VPC → Create VPC → VPC only.** Name: `VPC-B`. CIDR: `10.200.0.0/16`. Create.
3. **VPC → Internet Gateways → Create internet gateway.** Name: `VPC-B-igw`. Create, then **Actions → Attach to VPC → VPC-B**.

*Why VPC-B needs an Internet Gateway and VPC-A doesn't:* VPC-B is playing the role of your office network reaching out to the public internet — a real office has an ISP connection. VPC-A is playing the role of your protected internal AWS network, which should never need direct internet access; it only needs to be reachable through the tunnel.

### Step 5 — Create a public subnet in VPC-B, with a route to the internet

1. **VPC → Subnets → Create subnet.** VPC: `VPC-B`. Name: `VPC-B-public`. AZ: any. CIDR: `10.200.0.0/24`.
2. Enable auto-assign public IPv4 on this subnet (Actions → Edit subnet settings).
3. **VPC → Route Tables** → find (or create) the route table associated with `VPC-B-public`.
4. Add a route: destination `0.0.0.0/0` → target `VPC-B-igw`. Save.
5. Confirm this route table is associated with `VPC-B-public` (Subnet associations tab).

### Step 6 — Launch EC2-B in the public subnet, and give it an Elastic IP

1. **EC2 → Launch instance.** Name: `EC2-B`.
2. **Amazon Linux 2** (not 2023 — see Prerequisites #7), `t2.micro`.
3. Network settings → VPC: `VPC-B`, Subnet: `VPC-B-public`, Auto-assign public IP: **Enable**.
4. Create a new security group `EC2-B-SG` with:
   - Inbound: **SSH (22)**, source: your own IP (for management)
   - Inbound: **UDP 500** (IKE), source: `0.0.0.0/0` — AWS's tunnel endpoints need to reach this
   - Inbound: **UDP 4500** (IPsec NAT-Traversal), source: `0.0.0.0/0`
   - Inbound: **All ICMP – IPv4**, source `10.100.0.0/16`
   - Inbound: **Custom protocol — ESP (protocol 50)**, source: `0.0.0.0/0`
5. Launch.
6. **EC2 → Elastic IPs → Allocate Elastic IP address**, then **Actions → Associate** it to EC2-B.

*Why an Elastic IP specifically:* a regular auto-assigned public IP can change if the instance stops and starts. AWS's Customer Gateway configuration is pinned to one specific public IP — if that IP ever changed, the tunnel would break and you'd have to reconfigure it. An Elastic IP guarantees it never does, exactly like a real office's static WAN IP from their ISP.

*Why UDP 500/4500 and ESP specifically, and not just "allow VPN traffic":* those are the actual protocols IPsec uses — UDP 500 negotiates the tunnel (IKE), UDP 4500 carries traffic when NAT is involved, and ESP (protocol 50) is the encrypted payload itself. All three need to be open, or the tunnel negotiation fails silently.

### Step 7 — Enable IP forwarding and disable source/destination check on EC2-B

EC2-B needs to behave like a router — forwarding packets between the tunnel and the rest of VPC-B (and vice versa) — not just receive traffic addressed to itself.

1. **EC2 → select EC2-B → Actions → Networking → Change source/destination check → Disable.**
   *Why:* by default, AWS drops any packet that arrives at an instance addressed to something other than that instance's own IP, assuming it's a misconfiguration. Since EC2-B needs to receive traffic addressed to `10.100.0.0/16` and forward it onward, that default protection has to be turned off for this specific ENI.
2. SSH into EC2-B and set three kernel network settings — not just IP forwarding:
   ```bash
   sudo tee -a /etc/sysctl.conf <<'EOF'
   net.ipv4.ip_forward = 1
   net.ipv4.conf.default.rp_filter = 0
   net.ipv4.conf.default.accept_source_route = 0
   EOF
   sudo sysctl -p
   ```
   *Why all three, not just `ip_forward`:* `ip_forward=1` is the one people remember — it tells the kernel to forward packets between interfaces at all. But `rp_filter` (reverse-path filtering) is a separate anti-spoofing check that, left at its default, drops a forwarded packet whenever its source address doesn't match what the kernel expects for the interface it arrived on — which is exactly what happens here, since EC2-B is legitimately forwarding traffic for a different subnet than its own. `accept_source_route=0` is a hardening setting AWS's own downloaded configuration also asks for. Skipping any of the three can leave you with a tunnel that shows "UP" and still passes no traffic.

### Step 8 — Create the Customer Gateway (back in Mumbai)

1. Switch region back to **Mumbai (ap-south-1)**.
2. **VPC → Site-to-Site VPN Connections → Customer Gateways → Create Customer Gateway.**
3. Name: `CGW-EC2-B`. Routing: **Static**. IP address: EC2-B's **Elastic IP**. Leave BGP ASN as the default — it's ignored for static routing.
4. Create.

*What this object actually represents:* a Customer Gateway isn't a real device — it's just AWS's record of "here is the public IP of the router on the other end, and here's how we'll route to it." Nothing is provisioned yet; you're just registering an endpoint.

### Step 9 — Create the Virtual Private Gateway and attach it to VPC-A

1. **VPC → Virtual Private Gateways → Create Virtual Private Gateway.** Name: `VGW-A`. ASN: default (Amazon-generated). Create.
2. Select it → **Actions → Attach to VPC → VPC-A.**

*What this is:* the actual AWS-managed endpoint that will terminate the two IPsec tunnels on AWS's side. Attaching it to VPC-A is what lets VPC-A's route table eventually point traffic at it.

### Step 10 — Create the Site-to-Site VPN Connection

1. **VPC → Site-to-Site VPN Connections → Create VPN Connection.**
2. Name: `VPN-A-to-B`.
3. Target gateway type: **Virtual Private Gateway** → select `VGW-A`.
4. Customer gateway: **Existing** → select `CGW-EC2-B`.
5. Routing options: **Static**.
6. Static IP prefixes: enter `10.200.0.0/16` — this tells AWS "traffic for this range should go down this tunnel."
7. Leave tunnel options (pre-shared key, inside CIDR) on **Amazon generated** — don't invent your own for a first pass.
8. Create. It will sit in **pending** status for a couple of minutes while AWS provisions both tunnels — this is normal.

### Step 11 — Enable route propagation (or add the static route) in VPC-A

1. **VPC → Route Tables** → find the route table associated with `VPC-A-private`.
2. **Route Propagation tab → Edit route propagation → enable propagation from `VGW-A`.**

*What this does, and why it's needed even though you already told AWS the static prefix in Step 10:* Step 10 told the **VPN connection** which traffic belongs on the tunnel. Route propagation is what pushes that knowledge into VPC-A's actual **route table**, so instances in VPC-A know to send return traffic for `10.200.0.0/16` back out through the VGW. Without this step, the tunnel will show "UP," but nothing will actually be reachable — a very common point people get stuck at.

*(Alternative if you want to see it explicitly instead of relying on propagation: add a manual static route, destination `10.200.0.0/16`, target the VGW.)*

### Step 12 — Download the tunnel configuration file

1. Select the VPN connection → **Download Configuration.**
2. Vendor: choose **Libreswan** if it's listed; if only **Openswan** appears, pick that instead — the generated config format targets the same `ipsec.conf`/`ipsec.secrets` syntax Libreswan uses, so it's the correct choice either way.
3. Platform: leave default. Software version: leave default.
4. Download and open the file. You'll see, for **each of the two tunnels**: the AWS-side outside IP address, the pre-shared key, and the inside tunnel CIDR (a small `/30` used only for the tunnel's own routing, not your real network CIDRs).

Keep this file open — you'll copy values from it into EC2-B in the next step.

### Step 13 — Install and configure Libreswan on EC2-B

1. SSH into EC2-B.
2. Install Libreswan:
   ```bash
   sudo yum install -y libreswan
   sudo systemctl start ipsec
   ```
3. Open `/etc/ipsec.conf` and confirm this line is present and **not** commented out:
   ```
   include /etc/ipsec.d/*.conf
   ```
   *Why:* without this line, Libreswan never reads any tunnel configuration files you drop into `/etc/ipsec.d/` — the service will start successfully but silently have zero tunnels configured, which looks identical to a networking problem if you don't know to check this first.
4. Create `/etc/ipsec.d/aws-vpn.conf` with one `conn` block per tunnel (repeat for tunnel 2 with its own values):
   ```
   conn Tunnel1
     authby=secret
     auto=start
     left=%defaultroute
     leftid=<EC2-B Elastic IP>
     right=<Tunnel 1 AWS outside IP from the downloaded file>
     type=tunnel
     ikelifetime=8h
     keylife=1h
     phase2alg=aes128-sha1;modp1024
     ike=aes128-sha1;modp1024
     keyingtries=%forever
     leftsubnet=10.200.0.0/16
     rightsubnet=10.100.0.0/16
     dpddelay=10
     dpdtimeout=30
     dpdaction=restart_by_peer
   ```
   *Explaining the non-obvious lines:* `leftsubnet`/`rightsubnet` are what actually make this a **Site-to-Site** (network-to-network) tunnel rather than a point-to-point one — they tell Libreswan "everything in `10.200.0.0/16` on my side may talk to everything in `10.100.0.0/16` on the other side," not just this one host. `dpd*` settings are dead-peer-detection — how the tunnel notices the other side went away and restarts itself.
5. Add the pre-shared key in `/etc/ipsec.secrets`, one line per tunnel:
   ```
   <EC2-B Elastic IP> <Tunnel 1 AWS outside IP>: PSK "<pre-shared key from downloaded file>"
   ```
6. Enable the service on boot and restart it so the new config and secrets are picked up:
   ```bash
   sudo systemctl enable ipsec
   sudo systemctl restart ipsec
   ```

### Step 14 — Add EC2-B as a router in VPC-B's route table

1. **VPC → Route Tables** (in N. Virginia) → the route table associated with `VPC-B-public`.
2. Add route: destination `10.100.0.0/16` → target: **Instance** → select `EC2-B`.

*Why this is needed:* Step 7 made EC2-B *capable* of forwarding traffic, but nothing yet tells VPC-B's network *to send* `10.100.0.0/16`-bound traffic to EC2-B in the first place. This route is VPC-B's equivalent of Step 11 on the AWS side — without it, packets from anything else you later add in VPC-B would have no path toward VPC-A at all. (For this lab, EC2-B is the only host in VPC-B, so this step matters most if you extend the lab with more hosts later — but set it up correctly now regardless.)

### Step 15 — Verify the tunnels are up

1. Back in the Mumbai console, **VPC → Site-to-Site VPN Connections → select `VPN-A-to-B` → Tunnel Details tab.**
2. You want to see at least one (ideally both) tunnels showing status **UP**.
3. Cross-check from EC2-B:
   ```bash
   sudo ipsec status
   ```
   Look for the tunnel connection names showing as established (`IPsec SA established`).

### Step 16 — Test end-to-end connectivity

From **EC2-B**, ping and SSH to **EC2-A's private IP** (find it in the EC2-A console detail page):
```bash
ping 10.100.0.X
ssh -i your-key.pem ec2-user@10.100.0.X
```

If both succeed, you've proven the entire point of the lab: a host with **no public IP at all**, sitting in a private subnet, is reachable from a genuinely separate network — over the public internet — with everything in between encrypted.

---

## Part 4 — If it doesn't work: troubleshooting in the right order

Work through these in sequence — each one rules out a whole category of failure before you move to the next.

| Check | How | What it tells you |
|---|---|---|
| 1. Tunnel status in AWS console | Tunnel Details tab | If DOWN: the problem is between EC2-B and AWS — check security group (UDP 500/4500/ESP), check the Elastic IP is correctly entered in the Customer Gateway, check `ipsec.secrets` matches the downloaded PSK exactly. |
| 2. `sudo ipsec status` on EC2-B | SSH into EC2-B | If it shows no established SA: check `ipsec.conf` syntax, check `systemctl status ipsec` for errors, confirm the AWS outside IP in the config matches what's currently shown in the console (it can occasionally differ from an old download). |
| 3. Route propagation in VPC-A | Route Tables → Routes tab | If `10.200.0.0/16` isn't listed with target `vgw-...`: propagation wasn't actually enabled, or the route table checked isn't the one associated with EC2-A's subnet. |
| 4. Route to EC2-B in VPC-B | Route Tables → Routes tab | If `10.100.0.0/16` isn't listed with target `EC2-B`'s instance ID: Step 14 was missed. |
| 5. Source/dest check on EC2-B | EC2 → Networking tab | Must show "Disabled." If enabled, EC2-B silently drops forwarded traffic. |
| 6. Kernel network settings on EC2-B | `sysctl net.ipv4.ip_forward net.ipv4.conf.default.rp_filter net.ipv4.conf.default.accept_source_route` | `ip_forward` must return `1`; the other two must return `0`. If `ip_forward` is `0`, the OS drops forwarded packets outright. If `rp_filter` is nonzero, it silently drops forwarded packets whose source doesn't match the interface's expected route — the more common, sneakier version of this failure since the tunnel and everything else can look correct. |
| 7. Security group on EC2-A | EC2-A-SG inbound rules | Must allow ICMP and SSH from `10.200.0.0/16` specifically — a common mistake is allowing from `0.0.0.0/0` on EC2-B's side but forgetting EC2-A also needs its own inbound rule. |
| 8. Only one tunnel UP | Tunnel Details tab | Not necessarily a problem — one tunnel carrying traffic while the second sits idle/standby is normal and expected in static routing without any extra failover logic. |

---

## Part 5 — Cleanup (in dependency order, to avoid stuck-resource errors)

Delete in exactly this order — AWS won't let you delete something that's still attached to or referenced by something else.

1. **Delete the Site-to-Site VPN Connection** (`VPN-A-to-B`).
2. **Detach the Virtual Private Gateway** from VPC-A, then delete `VGW-A`.
3. **Delete the Customer Gateway** (`CGW-EC2-B`).
4. **Terminate EC2-A and EC2-B.**
5. **Release the Elastic IP** from EC2-B (it bills hourly if left allocated but unattached).
6. **Delete the route table entries / route tables**, subnets, and Internet Gateway (detach first, then delete) in VPC-B.
7. **Delete VPC-A and VPC-B.**

Skipping the release of the Elastic IP is the single most common leftover cost from this lab — it keeps billing even with nothing attached to it.

---

## Part 6 — Billing variables in this specific lab

This lab uses a **VGW-based, static-routing** VPN connection — the billing picture is different (and simpler) than a Transit-Gateway-based setup, so don't assume TGW pricing applies here.

| Billing variable | Why it applies here | Notes |
|---|---|---|
| **VPN connection — hourly rate** | One Site-to-Site VPN connection (`VPN-A-to-B`) | Billed per connection-hour it exists and is **available**, regardless of tunnel UP/DOWN state or how much traffic passes through it. This is the charge most likely to be forgotten after the lab. |
| **Data transfer OUT** | Traffic leaving AWS through the tunnel toward EC2-B | Standard EC2 data-transfer-out rates. **Data coming INTO AWS over the VPN is free** — only the outbound direction is billed. For a ping/SSH test this is negligible, but matters for real workloads. |
| **Public IPv4 address charges** | Every VPN tunnel uses a public IPv4 address on the AWS side | AWS charges for public IPv4 addresses generally (not just unattached ones) — this applies to the addresses behind your tunnels too. |
| **Elastic IP on EC2-B** | Allocated to simulate a static "on-prem" public IP | Same public IPv4 charge as above — bills whether or not it's actively passing traffic, as long as it's allocated. |
| **EC2-A and EC2-B instance costs** | Two running instances, in two separate regions | Not a VPN-specific charge, but part of the lab's real total cost — standard EC2 on-demand hourly rate × 2 instances. |
| **Virtual Private Gateway (VGW)** | Attached to VPC-A | **No separate hourly charge** — unlike Transit Gateway, VGW attachment itself is free. The VPN connection's own hourly rate is the only gateway-side cost. |
| **Customer Gateway** | Registered as `CGW-EC2-B` | **No charge** — it's only a configuration record of EC2-B's public IP, not a provisioned resource. |

**One thing worth flagging if you ever rebuild this lab on Transit Gateway instead of VGW:** TGW adds its own hourly attachment charge plus a separate per-GB data-processing charge, on top of the VPN connection's own hourly rate — costs stack in a way they don't with the simpler VGW setup used here.

---

## Part 7 — Upgrading to Transit Gateway (TGW)

This part picks up exactly where Part 6 ends. You have a working VGW-based Site-to-Site VPN. Now you'll replace the VGW with a **Transit Gateway** — the AWS network hub designed for multi-VPC, multi-site architectures.

### 7.1 Why switch from VGW to Transit Gateway?

Before touching anything, understand the problem TGW solves so you don't swap for no reason.

**The VGW limitation — it only serves one VPC:**

```
Current (VGW-based):

EC2-B (on-prem) ──[VPN tunnel]──► VGW ──► VPC-A only
                                   │
                                   └── Cannot reach VPC-C or VPC-D
                                       (you'd need a separate VPN per VPC!)
```

**With Transit Gateway — one hub serves everything:**

```
Upgraded (TGW-based):

                              ┌─────────────────────────────────────────────┐
                              │           TRANSIT GATEWAY (TGW)             │
                              │                                             │
EC2-B (on-prem) ──[VPN]──────►│  VPN Attachment                            │
                              │       │                                     │
                              │       ├──► VPC-A Attachment ──► VPC-A       │
                              │       │    (10.100.0.0/16)    (EC2-A)       │
                              │       │                                     │
                              │       ├──► VPC-C Attachment ──► VPC-C       │
                              │       │    (10.150.0.0/16)    (future app)  │
                              │       │                                     │
                              │       └──► VPC-D Attachment ──► VPC-D       │
                              │            (10.160.0.0/16)    (databases)   │
                              └─────────────────────────────────────────────┘

One VPN connection → reaches ALL attached VPCs automatically.
```

**When the upgrade makes sense:**

| Situation | Stay with VGW | Switch to TGW |
|---|---|---|
| Single VPC, single on-prem site | ✅ Simpler, cheaper | Overkill |
| 2+ VPCs need on-prem access | ❌ Need one VGW + one VPN per VPC | ✅ One TGW, one VPN |
| VPC-to-VPC traffic needed | ❌ Not possible via VGW | ✅ TGW routes between VPCs |
| Multiple branch offices (3+ sites) | Complex mesh | ✅ All connect to one TGW hub |
| Need centralised egress inspection | Hard | ✅ Route all traffic through one "inspection VPC" |

---

### 7.2 Architecture of what you'll build

```
Mumbai Region (ap-south-1)
─────────────────────────────────────────────────────────────────────────────
 ┌──────────────────────────────────────────────────────────────────────────┐
 │                        TRANSIT GATEWAY (TGW-Lab)                        │
 │                      ASN: 64512 (Amazon default)                        │
 │                                                                          │
 │   ┌───────────────────┐   ┌───────────────────┐   ┌──────────────────┐  │
 │   │  VPC-A Attachment │   │  VPN Attachment   │   │ (future VPC-C)   │  │
 │   │  10.100.0.0/16    │   │  to CGW-EC2-B     │   │  add anytime     │  │
 │   └────────┬──────────┘   └────────┬──────────┘   └──────────────────┘  │
 └────────────┼─────────────────────── ┼─────────────────────────────────── ┘
              │                        │
              ▼                        │  (two IPsec tunnels over internet)
         ┌─────────┐                   │
         │  VPC-A  │                   ▼
         │  EC2-A  │        ┌──────────────────────┐
         │(private)│        │  N. Virginia (us-east-1)
         └─────────┘        │  VPC-B / EC2-B        │
                            │  Elastic IP: x.x.x.x  │
                            │  Libreswan (IPsec)     │
                            └──────────────────────┘

Key change from Part 3:
  OLD: EC2-B ──[VPN]──► VGW ──► VPC-A
  NEW: EC2-B ──[VPN]──► TGW ──► VPC-A (and any future VPCs)
```

---

### 7.3 Step-by-step — upgrade to Transit Gateway

**All steps in Mumbai (ap-south-1) unless noted otherwise.**

---

#### Step TGW-1 — Delete the old VPN connection and detach the VGW

You cannot attach a VGW and a TGW to the same VPN connection — you need to build a fresh VPN connection targeting the TGW. Do this in dependency order:

1. **VPC → Site-to-Site VPN Connections** → select `VPN-A-to-B` → **Actions → Delete VPN Connection**. Confirm. (Takes ~1 minute.)
2. **VPC → Virtual Private Gateways** → select `VGW-A` → **Actions → Detach from VPC**. Wait for detachment to complete.
3. **Actions → Delete Virtual Private Gateway**. Confirm.

> ⚠️ **Heads up on downtime**: deleting the VPN connection drops both tunnels immediately. EC2-A becomes unreachable from EC2-B from this point until the new TGW-based VPN is fully configured and tunnels are back UP. Plan for ~15–20 minutes of downtime in a lab; in production you'd keep the old connection alive until the new one is tested.

---

#### Step TGW-2 — Create the Transit Gateway

1. **VPC → Transit Gateways → Create Transit Gateway**.
2. Fill in:
   - **Name tag**: `TGW-Lab`
   - **Description**: (optional) `Lab TGW replacing VGW-A`
   - **Amazon side ASN**: leave as `64512` (Amazon default) — only matters for BGP; fine for our static-routing lab
   - **DNS support**: Enabled ✅
   - **VPN ECMP support**: Enabled ✅ (allows active-active across tunnels if you later add BGP)
   - **Default route table association**: Enabled ✅ (new attachments automatically associate with the TGW default route table)
   - **Default route table propagation**: Enabled ✅ (new attachments automatically propagate their routes into the default route table)
   - **Multicast support**: Disabled (not needed)
3. Click **Create Transit Gateway**.
4. Wait for **State** to change from `pending` to `available` — usually 2–3 minutes. Refresh the list.

> 💡 **What "default route table association/propagation" does**: when you attach VPC-A and the VPN to the TGW, their routes are automatically added to the TGW's default route table, and they automatically get associated with it. This means TGW will know how to route between them without you manually editing the TGW route table — correct for this lab. In production you'd often disable these defaults and create per-attachment route tables for isolation.

---

#### Step TGW-3 — Attach VPC-A to the Transit Gateway

1. **VPC → Transit Gateway Attachments → Create Transit Gateway Attachment**.
2. Fill in:
   - **Transit Gateway ID**: select `TGW-Lab`
   - **Attachment type**: `VPC`
   - **VPC ID**: select `VPC-A`
   - **Subnet IDs**: select `VPC-A-private` (the subnet where EC2-A lives)
     > You must specify at least one subnet per AZ you want TGW to be reachable from. For a single-AZ lab, one subnet is enough.
   - **DNS support**: Enabled
   - **IPv6 support**: Disabled
3. Click **Create Transit Gateway Attachment**.
4. Wait for **State** to reach `available`.

**What just happened**: TGW placed an elastic network interface (ENI) inside `VPC-A-private`. Traffic from EC2-A destined for the TGW will now flow to that ENI — but only once you update the VPC-A route table in Step TGW-5.

---

#### Step TGW-4 — Create a new VPN Connection targeting the TGW

1. **VPC → Site-to-Site VPN Connections → Create VPN Connection**.
2. Fill in:
   - **Name**: `VPN-TGW-to-EC2B`
   - **Target gateway type**: `Transit Gateway` → select `TGW-Lab`
     > *(This is the only field that's different from the original VGW-based setup — everything else is identical.)*
   - **Customer Gateway**: `Existing` → select `CGW-EC2-B` (same one as before — EC2-B's Elastic IP hasn't changed)
   - **Routing options**: `Static`
   - **Static IP prefixes**: `10.200.0.0/16` (the VPC-B / on-prem CIDR — same as before)
   - **Tunnel options**: leave as Amazon generated
3. Click **Create VPN Connection**.
4. Wait for **State** to reach `available` (the tunnels themselves will show `DOWN` until EC2-B is reconfigured — this is expected).
5. **Download the new configuration file**: select the new VPN connection → **Download Configuration** → Vendor: Libreswan (or Openswan) → Download.

> ⚠️ **Do not reuse the old PSKs or AWS outside IPs.** The new connection has brand-new pre-shared keys and brand-new AWS-side public IPs for each tunnel. The downloaded config file for this connection contains the correct new values — use those.

**What TGW did automatically** (because you enabled default route table association/propagation in Step TGW-2):
- Created a **TGW VPN attachment** for this VPN connection
- Added a route for `10.200.0.0/16` into the TGW default route table, pointing at this VPN attachment
- Added a route for `10.100.0.0/16` into the TGW default route table, pointing at the VPC-A attachment from Step TGW-3

This means TGW now knows how to route between VPC-A and the on-prem network — **before you've touched a single VPC route table**.

---

#### Step TGW-5 — Update the VPC-A route table to point at TGW instead of VGW

The VPC-A route table previously pointed `10.200.0.0/16` at the VGW (or had route propagation from the VGW). The VGW is gone — update it to point at the TGW.

1. **VPC → Route Tables** → select the route table associated with `VPC-A-private`.
2. **Routes tab → Edit routes**.
3. If there's an old route `10.200.0.0/16 → vgw-...`: **delete it**.
4. **Add route**:
   - Destination: `10.200.0.0/16`
   - Target: **Transit Gateway** → select `TGW-Lab`
5. Save changes.

> 💡 **Route propagation note**: with VGW, you could enable "route propagation" to auto-add routes. TGW does not support propagating routes *into VPC route tables* — you always add VPC-side routes manually (or via automation). The propagation you enabled in TGW-2 only affects the **TGW's own internal route table**, not VPC route tables.

**What the route table looks like now:**

```
VPC-A-private Route Table:
┌──────────────────────┬──────────────────────────────────────┐
│ Destination          │ Target                               │
├──────────────────────┼──────────────────────────────────────┤
│ 10.100.0.0/16        │ local                                │
│ 10.200.0.0/16        │ tgw-xxxxxxxxxxxxxxxxx  (TGW-Lab) ✅  │
└──────────────────────┴──────────────────────────────────────┘
```

---

#### Step TGW-6 — Verify the TGW route table (optional but educational)

1. **VPC → Transit Gateway Route Tables** → select the default TGW route table.
2. **Routes tab** — you should see:

```
TGW Default Route Table:
┌──────────────────────┬────────────────────────────────────┬───────────┐
│ CIDR                 │ Attachment                         │ Type      │
├──────────────────────┼────────────────────────────────────┼───────────┤
│ 10.100.0.0/16        │ vpc-attach-xxxxx (VPC-A)           │ propagated│
│ 10.200.0.0/16        │ vpn-attach-xxxxx (VPN-TGW-to-EC2B) │ propagated│
└──────────────────────┴────────────────────────────────────┴───────────┘
```

Both routes were added automatically by TGW because of the default propagation settings. This is what allows traffic to flow between VPC-A and the on-prem network through TGW — no manual TGW route editing needed for this lab.

---

#### Step TGW-7 — Reconfigure Libreswan on EC2-B with the new tunnel values

The new VPN connection has different AWS-side public IPs and different pre-shared keys. EC2-B's Libreswan configuration still points at the old, deleted VPN connection's endpoints — it must be updated.

**SSH into EC2-B (N. Virginia)** and do the following:

**1. Stop the IPsec service:**
```bash
sudo systemctl stop ipsec
```

**2. Open the downloaded configuration file** for the new VPN connection. Find, for each tunnel:
- `Outside IP address` (the AWS-side public IP for that tunnel)
- `Pre-Shared Key`

**3. Replace `/etc/ipsec.d/aws-vpn.conf` with the new values:**
```bash
sudo tee /etc/ipsec.d/aws-vpn.conf <<'EOF'
conn Tunnel1
  authby=secret
  auto=start
  left=%defaultroute
  leftid=<EC2-B Elastic IP>                    # same as before
  right=<NEW Tunnel 1 AWS outside IP>           # from new downloaded config
  type=tunnel
  ikelifetime=8h
  keylife=1h
  phase2alg=aes128-sha1;modp1024
  ike=aes128-sha1;modp1024
  keyingtries=%forever
  leftsubnet=10.200.0.0/16
  rightsubnet=10.100.0.0/16
  dpddelay=10
  dpdtimeout=30
  dpdaction=restart_by_peer

conn Tunnel2
  authby=secret
  auto=start
  left=%defaultroute
  leftid=<EC2-B Elastic IP>                    # same as before
  right=<NEW Tunnel 2 AWS outside IP>           # from new downloaded config
  type=tunnel
  ikelifetime=8h
  keylife=1h
  phase2alg=aes128-sha1;modp1024
  ike=aes128-sha1;modp1024
  keyingtries=%forever
  leftsubnet=10.200.0.0/16
  rightsubnet=10.100.0.0/16
  dpddelay=10
  dpdtimeout=30
  dpdaction=restart_by_peer
EOF
```

**4. Replace `/etc/ipsec.secrets` with the new pre-shared keys:**
```bash
sudo tee /etc/ipsec.secrets <<'EOF'
<EC2-B Elastic IP> <NEW Tunnel 1 AWS outside IP>: PSK "<new PSK for tunnel 1>"
<EC2-B Elastic IP> <NEW Tunnel 2 AWS outside IP>: PSK "<new PSK for tunnel 2>"
EOF
```

**5. Restart the IPsec service:**
```bash
sudo systemctl start ipsec
```

**6. Watch the tunnel establishment logs in real time:**
```bash
sudo journalctl -u ipsec -f
```
You should see IKE Phase 1 and Phase 2 negotiation messages appearing within 30–60 seconds. Ctrl+C to exit once you see `IPsec SA established`.

---

#### Step TGW-8 — Verify everything end-to-end

**Check 1 — Tunnel state in AWS Console (Mumbai):**
```
VPC → Site-to-Site VPN Connections → VPN-TGW-to-EC2B → Tunnel Details tab
→ At least one tunnel: Status = UP ✅
```

**Check 2 — Libreswan status on EC2-B:**
```bash
sudo ipsec status
# Look for: "IPsec SA established" for Tunnel1 and/or Tunnel2
```

**Check 3 — TGW route table has both routes:**
```
VPC → Transit Gateway Route Tables → default → Routes tab
→ 10.100.0.0/16 → VPC-A attachment ✅
→ 10.200.0.0/16 → VPN attachment ✅
```

**Check 4 — VPC-A route table points to TGW:**
```
VPC → Route Tables → VPC-A-private → Routes tab
→ 10.200.0.0/16 → tgw-xxxxxxxxx ✅
```

**Check 5 — End-to-end connectivity from EC2-B:**
```bash
# From EC2-B in N. Virginia, ping EC2-A's private IP:
ping 10.100.0.X

# SSH into EC2-A through the tunnel:
ssh -i your-key.pem ec2-user@10.100.0.X
```

If ping and SSH work, the TGW upgrade is complete. The traffic path is now:

```
EC2-B → [IPsec tunnel] → TGW VPN Attachment → TGW → TGW VPC-A Attachment → VPC-A subnet ENI → EC2-A
```

---

### 7.4 Extend the lab — add a second VPC through the same TGW

This is what makes TGW worth it. Add VPC-C without touching EC2-B's config at all:

**Step 1: Create VPC-C in Mumbai**
```
VPC → Create VPC → VPC only
Name: VPC-C, CIDR: 10.150.0.0/16
```

**Step 2: Create a subnet and an EC2 instance in VPC-C**
```
Subnet: 10.150.0.0/24 (private, no public IP on instances)
EC2-C: Amazon Linux 2023, private IP only, SG allows ICMP from 10.200.0.0/16
```

**Step 3: Attach VPC-C to TGW-Lab**
```
VPC → Transit Gateway Attachments → Create attachment
Type: VPC, TGW: TGW-Lab, VPC: VPC-C, Subnet: VPC-C-private
```
Wait for state → `available`. TGW automatically adds `10.150.0.0/16` to its route table.

**Step 4: Update VPC-C's route table**
```
Add route: 10.200.0.0/16 → tgw-xxxxxxxxx (TGW-Lab)
```

**Step 5: Tell EC2-B about the new subnet**

On EC2-B, update the Libreswan config to add `10.150.0.0/16` to the subnet the tunnel covers. The cleanest way is to widen `leftsubnet` to cover both:

```bash
# Edit /etc/ipsec.d/aws-vpn.conf
# Change leftsubnet line (both conn blocks) from:
#   rightsubnet=10.100.0.0/16
# to:
#   rightsubnet=10.0.0.0/8   ← covers 10.100.x and 10.150.x and any future 10.x.x.x
# Then restart:
sudo systemctl restart ipsec
```

**Step 6: Test from EC2-B — no AWS-side changes at all:**
```bash
ping 10.150.0.X   # EC2-C's private IP
```

This works because the TGW already knows `10.150.0.0/16 → VPC-C attachment` from the propagation in Step 3. The on-prem side (EC2-B) just needed to know the new subnet exists. No new VPN connections, no new CGWs, no changes to VPC-A.

---

### 7.5 TGW-specific troubleshooting

Problems that are unique to TGW (not present in the VGW setup):

| Symptom | Most likely cause | Fix |
|---|---|---|
| Tunnel is UP but VPC-A is unreachable | VPC-A route table still points at old VGW | Check Routes tab of VPC-A-private → must show `10.200.0.0/16 → tgw-...`, not `vgw-...` |
| Tunnel UP, VPC-A reachable, VPC-C not reachable | VPC-C route table has no route to TGW | Add `10.200.0.0/16 → tgw-...` to VPC-C's route table |
| TGW route table missing `10.100.0.0/16` | VPC-A attachment didn't propagate | Check TGW attachment state is `available`; check route table's propagation settings include the VPC-A attachment |
| TGW route table missing `10.200.0.0/16` | VPN attachment didn't propagate (rare) | Manually add a static route: `10.200.0.0/16 → VPN attachment` in the TGW route table |
| EC2-B can't reach EC2-C even though it can reach EC2-A | Libreswan `rightsubnet` doesn't include `10.150.0.0/16` | Widen `rightsubnet` on EC2-B to cover the new CIDR and restart ipsec |
| VPN connection creation fails with "target gateway not available" | TGW still in `pending` state | Wait for TGW state to reach `available` (2–3 minutes) before creating the VPN connection |
| Higher latency than before | Expected — TGW adds one extra routing hop compared to VGW | This is the trade-off for flexibility; not a bug |

---

### 7.6 Billing impact of the TGW upgrade

Switching from VGW to TGW changes the cost structure. Here's exactly what's added:

| Cost component | VGW setup (Part 6) | TGW setup (Part 7) |
|---|---|---|
| **VPN Connection hourly** | ✅ Billed | ✅ Billed (same rate) |
| **VGW attachment** | Free | N/A — VGW deleted |
| **TGW itself** | N/A | ✅ **NEW** — $0.05/hr per attachment |
| **TGW VPC-A attachment** | N/A | ✅ **NEW** — $0.05/hr |
| **TGW VPN attachment** | N/A | ✅ **NEW** — $0.05/hr |
| **TGW data processing** | N/A | ✅ **NEW** — $0.02/GB processed through TGW |
| **Data transfer OUT** | ✅ Billed | ✅ Billed (same rate) |
| **EC2 instances** | ✅ Billed | ✅ Billed (same) |
| **Elastic IP** | ✅ Billed | ✅ Billed (same) |

**Estimated monthly difference for this lab:**

```
VGW-based lab:
  VPN connection: $0.05 × 730 hrs        = $36.50
  Total extra (no VGW fee)               = $0.00
  Lab total (excl. EC2, EIP):            ~ $36.50/month

TGW-based lab:
  VPN connection: $0.05 × 730 hrs        = $36.50
  TGW VPC-A attachment: $0.05 × 730 hrs = $36.50
  TGW VPN attachment:   $0.05 × 730 hrs = $36.50
  TGW data processing:  negligible (lab) = ~$0.00
  Lab total (excl. EC2, EIP):            ~ $109.50/month

Extra cost for TGW in this lab: ~$73/month
```

> 💰 **When the cost is justified**: if you connect 5 VPCs instead of 1, the VGW alternative would need 5 separate VGWs + 5 VPN connections = $182.50/month just in VPN costs. TGW with 5 VPC attachments + 1 VPN attachment = $36.50 (VPN) + 6 × $36.50 (attachments) = $255/month — but you'd also save VPN connection costs for all 5 VPCs (5 × $36.50 = $182.50 saved on VPN connections). At scale, TGW wins on simplicity and often on cost too.

---

### 7.7 VGW vs Transit Gateway — decision reference

| Factor | Use VGW | Use Transit Gateway |
|---|---|---|
| **Number of VPCs** | 1 | 2 or more |
| **Number of on-prem sites** | 1 | 2 or more |
| **VPC-to-VPC routing** | Not possible via VGW | ✅ Native — TGW routes between all attached VPCs |
| **Centralised egress/inspection** | Not easy | ✅ Route all traffic through an "inspection VPC" |
| **Active/Active VPN (ECMP)** | Not supported | ✅ Supported (enables >1.25 Gbps effective throughput) |
| **Setup complexity** | Low | Medium |
| **Monthly base cost** | ~$36.50 (VPN only) | ~$109.50 (VPN + 2 attachments) |
| **Cost at 5+ VPCs** | Expensive (5 VGWs + 5 VPNs) | More efficient (1 TGW + attachments) |
| **AWS Direct Connect integration** | Separate, no shared hub | ✅ TGW is also the DX attachment point — one hub for all |

**Simple rule of thumb:**
- **≤ 1 VPC, 1 office, no plans to grow** → stay with VGW.
- **2+ VPCs, 2+ offices, or need VPC-to-VPC routing** → use Transit Gateway.

---

### 7.8 Cleanup — TGW resources (in dependency order)

When you're done with the TGW lab, delete in exactly this order:

1. **Delete the Site-to-Site VPN Connection** (`VPN-TGW-to-EC2B`).
2. **Delete the TGW VPC attachments** — go to **Transit Gateway Attachments**, select the VPC-A (and VPC-C if you added it) attachments → **Actions → Delete**.
3. Wait for attachments to reach `deleted` state (takes 1–2 minutes each).
4. **Delete the Transit Gateway** (`TGW-Lab`) → **Transit Gateways → Actions → Delete**. This will fail if any attachments still exist — that's why step 2 comes first.
5. **Delete the Customer Gateway** (`CGW-EC2-B`).
6. **Terminate EC2-A and EC2-B.**
7. **Release the Elastic IP** (releases immediately, stops billing).
8. **Delete subnets, route tables, Internet Gateway** (detach first), then **delete VPC-A and VPC-B** (and VPC-C if created).

> ⏱️ The TGW itself can take 3–5 minutes to fully delete after you confirm. If you try to delete the Customer Gateway before TGW deletion finishes, it may fail — wait for TGW state to reach `deleted` first.
