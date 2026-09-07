# AWS Site-to-Site VPN — End-to-End Hands-On Lab

Simulating a real Site-to-Site VPN entirely inside AWS: one VPC plays the role of your **AWS-side production network**, a second VPC (in a different region, so it's genuinely "remote") plays the role of your **on-premises / corporate network**, with an EC2 instance running IPsec software standing in for your physical office router.

---

## Part 1 — Understand the architecture first

Before touching the console, know exactly what you're building and why each piece exists.

<img width="1831" height="859" alt="image" src="https://github.com/user-attachments/assets/e4c11616-9f70-4dc4-af1b-c75925df2cd0" />


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
