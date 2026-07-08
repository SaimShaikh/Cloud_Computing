# AWS Site-to-Site VPN — Mumbai (AWS Side) ↔ Singapore (Simulated On-Prem) Lab Guide

---

## 1. Scenario

You are simulating a real enterprise hybrid-connectivity setup using two AWS regions instead of a real data center:

- **Mumbai (ap-south-1)** = plays the role of your **AWS cloud environment**. It hosts a VPC, an EC2 instance (your "cloud workload"), a **Virtual Private Gateway (VGW)**, a **Customer Gateway (CGW)** object, and the VPN Connection itself.
- **Singapore (ap-southeast-1)** = plays the role of your **on-premises data center / branch office**. It hosts its own VPC, Internet Gateway, subnet, route table, and a single EC2 instance that acts as the **on-prem VPN device** (like a Cisco ASA or a physical router would in real life), running **Openswan** to terminate the IPSec tunnel.

Goal: establish an encrypted **Site-to-Site VPN (IPSec)** tunnel between the two regions so that the Mumbai EC2 (private, AWS side) and the Singapore EC2 (public, "on-prem" side) can reach each other over their private IPs, exactly like a real HQ-to-cloud VPN.

This lab is **console-first** (every clickable step is a console step). AWS CLI commands are provided separately in the **CLI Reference** section for people who want to replicate it as Infrastructure-as-Code later.

---

## 2. Concept Deep Dive

### 2.1 What is a Site-to-Site VPN?

An AWS Site-to-Site VPN creates an encrypted **IPSec tunnel** over the public internet between:
- Your **on-premises network** (represented by a **Customer Gateway**, a physical/virtual device with a static public IP), and
- Your **AWS VPC** (represented by a **Virtual Private Gateway** attached to that VPC, or a **Transit Gateway**).

Unlike Direct Connect (a private, dedicated physical circuit), a Site-to-Site VPN rides on the **public internet** but the payload is encrypted using **IKE (Internet Key Exchange)** for key negotiation and **IPSec (ESP)** for encrypting the actual data.

Each AWS VPN Connection gives you **two tunnels** (Tunnel1 and Tunnel2) terminating on two different AWS public IPs, for redundancy — if one AWS endpoint has maintenance or an outage, the second tunnel keeps traffic flowing.

### 2.2 Why use Site-to-Site VPN?

- Extend your on-prem network into a VPC (hybrid cloud) without needing a physical leased line.
- Fast to provision (minutes) compared to Direct Connect (weeks/months of lead time).
- Good for backup connectivity — commonly used **as a failover path** for Direct Connect.
- Enables **lift-and-shift** migrations where some servers stay on-prem and some move to AWS but must talk to each other privately.
- Useful for **multi-site connectivity** when combined with Transit Gateway (hub-and-spoke).

### 2.3 Benefits

| Benefit | Detail |
|---|---|
| Fast provisioning | Tunnel is usually up within 5–10 minutes of creation |
| No physical hardware needed at AWS end | AWS manages the VPN endpoint infrastructure |
| Built-in redundancy | 2 tunnels per VPN connection, each on a different AWS endpoint |
| Encryption in transit | IKEv1/IKEv2 + IPSec ESP protects data over the public internet |
| Pay-as-you-go | No upfront cost, hourly + data transfer pricing |
| Works with Transit Gateway | Can fan out to many VPCs/on-prem sites from one connection |

### 2.4 Limitations

- **Throughput ceiling**: each tunnel is capped around **1.25 Gbps**. To scale beyond that you need multiple tunnels with ECMP (only possible via Transit Gateway, not a lone VGW) or Direct Connect.
- **Internet-dependent latency/jitter**: since it rides the public internet, performance is not as predictable as Direct Connect.
- Classic (non-accelerated) VPN does **not support ECMP load balancing across tunnels** on a VGW — Tunnel2 is standby/failover only, not active-active, unless attached to a Transit Gateway with ECMP enabled.
- Requires a **static public IP** on the customer-gateway side; no support for dynamic IPs unless behind additional NAT/DDNS tricks.
- BGP (dynamic routing) adds complexity; static routing (used in this lab) means routes must be manually maintained if CIDRs change.

### 2.5 Cost

- **VPN Connection hourly charge**: billed per hour the VPN connection exists (~$0.05/hr in most regions, check current [AWS VPC pricing page](https://aws.amazon.com/vpn/pricing/) for exact ap-south-1/ap-southeast-1 rates).
- **Data transfer OUT** from AWS to internet is billed at standard EC2 data transfer rates (data flowing through the tunnel is "data out" from AWS's perspective).
- EC2 instance cost for the Singapore "on-prem simulator" instance (t2.micro/t3.micro is enough for lab — free-tier eligible in most accounts).
- No extra charge for VGW or CGW objects themselves — you only pay for the VPN Connection resource and underlying EC2/data transfer.
- **Remember to delete the VPN Connection + terminate EC2 instances after the lab** — hourly VPN charges continue to accrue as long as the resource exists, even if the tunnel is DOWN.

### 2.6 Edge Cases

- If the Singapore EC2 instance is stopped/started (not just rebooted), its **public IP changes** (unless you assigned an Elastic IP) — this breaks the Customer Gateway registration since AWS expects a static IP. **Always use an Elastic IP** for the CGW-side instance.
- **Source/Destination Check** must be **disabled** on the Singapore EC2 instance's ENI — by default, AWS drops traffic where the EC2 is not the actual source/destination of the packet. Since this instance is acting as a router/gateway (encapsulating/decapsulating other traffic), this check must be turned off.
- If both tunnels show **DOWN** in the console but your Openswan config looks correct, the most common cause is a **Security Group** on the Singapore EC2 not allowing UDP 500, UDP 4500, and ESP (protocol 50) inbound from the AWS tunnel endpoint IPs.
- **Overlapping CIDRs** between Mumbai VPC and Singapore VPC will silently break routing — plan non-overlapping ranges before you start (this lab uses `10.10.0.0/16` for Mumbai and `10.20.0.0/16` for Singapore).
- Route propagation must be **explicitly enabled** on the Mumbai route table — a very common miss that leaves the VPN "UP" in console but unreachable in practice.
- `openswan` is only available on **Amazon Linux 2** (RHEL/CentOS 7-era package). Amazon Linux 2023 replaced it with **Libreswan** — if you launch an AL2023 instance the `yum install openswan` command in this lab will fail. Use **Amazon Linux 2 AMI** for the Singapore instance to match the commands given.

### 2.7 Assumptions

- You have an AWS account with permissions to create VPCs, EC2, VPN Gateways in both `ap-south-1` (Mumbai) and `ap-southeast-1` (Singapore).
- You are comfortable with basic Linux terminal usage (`vi`/`nano`, `sudo`).
- Mumbai EC2 uses **Amazon Linux 2** (any AMI works for this side, it's just the "cloud workload" being reached).
- Singapore EC2 **must** be **Amazon Linux 2** (openswan package requirement, see edge cases).
- No Direct Connect, Transit Gateway, or BGP is used in this lab — this is a **static-routing, VGW-based** classic Site-to-Site VPN, matching the exact Openswan config style you already have.
- CIDR plan:
  - Mumbai VPC: `10.10.0.0/16` → subnet `10.10.1.0/24`
  - Singapore VPC: `10.20.0.0/16` → subnet `10.20.1.0/24`

---

## 3. Architecture Diagram

```
                              PUBLIC INTERNET
                     ┌───────────────────────────────┐
                     │                                │
        ┌────────────┴───────────┐        ┌───────────┴────────────┐
        │   MUMBAI (ap-south-1)  │        │ SINGAPORE (ap-southeast-1)│
        │   "AWS Cloud Side"     │        │   "On-Prem Simulator"    │
        │                        │        │                          │
        │  VPC 10.10.0.0/16      │        │  VPC 10.20.0.0/16        │
        │  Subnet 10.10.1.0/24   │        │  Subnet 10.20.1.0/24     │
        │                        │        │                          │
        │  ┌──────────────────┐  │        │  ┌────────────────────┐  │
        │  │  EC2 (private)   │  │        │  │  EC2 + Elastic IP  │  │
        │  │  10.10.1.x       │  │        │  │  10.20.1.x         │  │
        │  │  "cloud workload"│  │        │  │  runs Openswan     │  │
        │  └────────┬─────────┘  │        │  │  = Customer Gateway│  │
        │           │            │        │  └─────────┬──────────┘  │
        │     Route Table        │        │             │             │
        │     (VGW propagation)  │        │        Route Table        │
        │           │            │        │             │             │
        │  ┌────────┴─────────┐  │        │      Internet Gateway     │
        │  │ Virtual Private  │◄─┼── IPSec Tunnel 1 & 2 ──┤            │
        │  │ Gateway (VGW)    │  │  (encrypted, over internet)          │
        │  └────────┬─────────┘  │        └──────────────────────────┘
        │           │            │
        │   VPN Connection       │
        │   Customer Gateway◄────┼─── registered with Singapore EIP
        └────────────────────────┘
```

---

## 4. Prerequisites

- Two Elastic IP-capable regions selected: `ap-south-1` (Mumbai), `ap-southeast-1` (Singapore).
- IAM permissions: `ec2:*` (VPC, Subnet, EC2, VPN Gateway, Customer Gateway, VPN Connection, Route Table, Elastic IP, Security Group).
- Key pair created in **both** regions (or reuse if your account allows cross-region key import) to SSH into both EC2 instances.
- A terminal / SSH client (PuTTY or native terminal) to connect to both instances.

---

## 5. Lab Part A — Singapore Region (Build this FIRST)

We build Singapore first because Mumbai's Customer Gateway object needs Singapore's Elastic IP to already exist.

### Step A1 — Create VPC (Singapore)

1. Switch console region to **Asia Pacific (Singapore) ap-southeast-1**.
2. Go to **VPC console → Your VPCs → Create VPC**.
3. Select **VPC only**.
4. Name tag: `singapore-onprem-vpc`
5. IPv4 CIDR: `10.20.0.0/16`
6. Tenancy: Default
7. Click **Create VPC**.

### Step A2 — Create Subnet (Singapore)

1. **VPC console → Subnets → Create subnet**.
2. VPC: `singapore-onprem-vpc`
3. Subnet name: `singapore-public-subnet`
4. Availability Zone: `ap-southeast-1a`
5. IPv4 CIDR: `10.20.1.0/24`
6. Click **Create subnet**.
7. Select the subnet → **Actions → Edit subnet settings** → Enable **Auto-assign public IPv4 address** → Save.

### Step A3 — Create and Attach Internet Gateway (Singapore)

1. **VPC console → Internet Gateways → Create internet gateway**.
2. Name: `singapore-igw`
3. Create, then select it → **Actions → Attach to VPC** → choose `singapore-onprem-vpc`.

### Step A4 — Create Route Table (Singapore)

1. **VPC console → Route Tables → Create route table**.
2. Name: `singapore-public-rt`
3. VPC: `singapore-onprem-vpc`
4. Create.
5. Select it → **Routes tab → Edit routes → Add route**:
   - Destination: `0.0.0.0/0`
   - Target: `singapore-igw`
6. Save routes.
7. Go to **Subnet associations tab → Edit subnet associations** → select `singapore-public-subnet` → Save.

### Step A5 — Create Security Group (Singapore)

1. **EC2 console → Security Groups → Create security group**.
2. Name: `singapore-vpn-sg`
3. VPC: `singapore-onprem-vpc`
4. Inbound rules:
   - SSH (22) — Source: My IP
   - Custom UDP — Port 500 — Source: 0.0.0.0/0 *(IKE)*
   - Custom UDP — Port 4500 — Source: 0.0.0.0/0 *(NAT-Traversal)*
   - Custom protocol — ESP (protocol 50) — Source: 0.0.0.0/0
   - ICMP — All — Source: 10.10.0.0/16 *(so you can ping Mumbai)*
5. Outbound: leave default (all allowed).
6. Create security group.

> **Edge case reminder:** In a strict production setup you'd lock UDP 500/4500/ESP inbound sources down to the two AWS VPN tunnel endpoint public IPs (visible after you create the VPN connection), not `0.0.0.0/0`. For this lab, 0.0.0.0/0 keeps it simple; tighten it afterward if you want extra practice.

### Step A6 — Launch EC2 Instance (Singapore) — Amazon Linux 2

1. **EC2 console → Launch instance**.
2. Name: `singapore-onprem-gateway`
3. AMI: **Amazon Linux 2 AMI** (must be AL2, not AL2023 — see edge cases).
4. Instance type: `t2.micro` (free tier)
5. Key pair: select/create one for Singapore.
6. Network settings → Edit:
   - VPC: `singapore-onprem-vpc`
   - Subnet: `singapore-public-subnet`
   - Auto-assign public IP: Enable
   - Security group: `singapore-vpn-sg`
7. Launch instance.

### Step A7 — Allocate and Associate Elastic IP (Singapore)

1. **EC2 console → Elastic IPs → Allocate Elastic IP address** → Allocate.
2. Select the new EIP → **Actions → Associate Elastic IP address**.
3. Resource type: Instance → select `singapore-onprem-gateway` → Associate.
4. **Note this Elastic IP down** — this is your **Customer Gateway public IP**, needed in Mumbai.

### Step A8 — Disable Source/Destination Check (Singapore)

1. Select `singapore-onprem-gateway` instance → **Actions → Networking → Change source/destination check**.
2. Set to **Stopped/Disabled**.
3. Save.

> This is mandatory — without disabling this check, AWS drops any packet where this instance isn't the literal source/destination, which breaks IPSec encapsulated traffic passing through it.

---

## 6. Lab Part B — Mumbai Region (AWS Cloud Side)

### Step B1 — Create VPC (Mumbai)

1. Switch console region to **Asia Pacific (Mumbai) ap-south-1**.
2. **VPC console → Your VPCs → Create VPC**.
3. Select **VPC only**.
4. Name tag: `mumbai-cloud-vpc`
5. IPv4 CIDR: `10.10.0.0/16`
6. Create VPC.

### Step B2 — Create Subnet (Mumbai)

1. **VPC console → Subnets → Create subnet**.
2. VPC: `mumbai-cloud-vpc`
3. Name: `mumbai-private-subnet`
4. AZ: `ap-south-1a`
5. IPv4 CIDR: `10.10.1.0/24`
6. Create subnet.
7. *(Leave auto-assign public IP disabled — this subnet stays private; reachability will come only via VPN.)*

### Step B3 — Create Route Table (Mumbai)

1. **VPC console → Route Tables → Create route table**.
2. Name: `mumbai-vpn-rt`
3. VPC: `mumbai-cloud-vpc`
4. Create.
5. **Subnet associations tab → Edit subnet associations** → select `mumbai-private-subnet` → Save.
   *(Routes to the VGW will be added automatically via route propagation in Step B7 — do not add a manual route yet.)*

### Step B4 — Create Security Group (Mumbai)

1. **EC2 console → Security Groups → Create security group**.
2. Name: `mumbai-vpn-sg`
3. VPC: `mumbai-cloud-vpc`
4. Inbound rules:
   - ICMP — All — Source: `10.20.0.0/16` *(so Singapore can ping this instance)*
   - SSH (22) — Source: `10.20.0.0/16` *(optional, to SSH via the tunnel once it's up)*
5. Create.

### Step B5 — Launch EC2 Instance (Mumbai)

1. **EC2 console → Launch instance**.
2. Name: `mumbai-cloud-workload`
3. AMI: Amazon Linux 2 (or any, this side doesn't run Openswan).
4. Instance type: `t2.micro`.
5. Key pair: select/create one for Mumbai.
6. Network settings:
   - VPC: `mumbai-cloud-vpc`
   - Subnet: `mumbai-private-subnet`
   - Auto-assign public IP: **Disable** (this instance is meant to be private, reachable only via VPN)
   - Security group: `mumbai-vpn-sg`
7. Launch.

> Since this instance has no public IP, you will only be able to SSH into it later **through the tunnel from the Singapore instance**, or via **EC2 Instance Connect Endpoint / SSM Session Manager** if you want out-of-band access for troubleshooting. For this lab, connectivity is proven with `ping` from the Singapore side once the tunnel is up.

### Step B6 — Create Customer Gateway (Mumbai)

1. **VPC console → Customer Gateways → Create customer gateway**.
2. Name: `singapore-cgw`
3. BGP ASN: `65000` (default, unused since we're doing static routing)
4. IP address: paste the **Singapore Elastic IP** from Step A7.
5. Certificate ARN: leave blank.
6. Device: leave blank (optional label).
7. Create customer gateway.

### Step B7 — Create Virtual Private Gateway (Mumbai)

1. **VPC console → Virtual Private Gateways → Create virtual private gateway**.
2. Name: `mumbai-vgw`
3. ASN: Amazon default ASN.
4. Create.
5. Select it → **Actions → Attach to VPC** → choose `mumbai-cloud-vpc`.
6. Go to `mumbai-vpn-rt` (Step B3) → **Route propagation tab → Edit route propagation** → enable propagation for `mumbai-vgw` → Save.

### Step B8 — Create VPN Connection (Mumbai)

1. **VPC console → Site-to-Site VPN Connections → Create VPN Connection**.
2. Name: `mumbai-singapore-vpn`
3. Target gateway type: **Virtual Private Gateway** → select `mumbai-vgw`
4. Customer gateway: **Existing** → select `singapore-cgw`
5. Routing options: **Static**
6. Static IP prefixes: enter `10.20.0.0/16` *(Singapore VPC CIDR — the network reachable behind the customer gateway)*
7. Tunnel options: leave defaults (Auto-generated Pre-Shared Keys, AWS-assigned tunnel inside CIDRs) — or set your own PSK now if you want to hardcode it into the `.secrets` file later without downloading the config.
8. Create VPN Connection.
9. Wait a few minutes — status moves from `pending` → `available`. (Tunnels themselves will still show `DOWN` until Openswan is configured in Part C — this is expected.)

### Step B9 — Download Configuration (Mumbai)

1. Select the `mumbai-singapore-vpn` connection → **Actions → Download configuration**.
2. Vendor: **Openswan**
3. Platform: **Openswan**
4. Software: **Openswan 2.6.38+**
   *(If "Openswan" is not listed in your console's vendor dropdown, choose **Generic** instead — the values you need are identical, you just map them into the `conn Tunnel1` block manually instead of copy-pasting a pre-filled one.)*
5. Download.
6. Open the downloaded file — extract these values for **Tunnel #1** (you'll only configure one tunnel in this lab; Tunnel #2 is available for extra-credit HA practice):
   - **Virtual Private Gateway (outside/public) IP** → this is `right=` in Openswan config
   - **Pre-Shared Key** → this goes in `aws-vpn.secrets`
   - **Customer Gateway (outside) IP** → this is your Singapore Elastic IP, `left=`/`leftid=`
7. **Known issue to watch for:** the Openswan-vendor download commonly includes a line `auth=esp` inside the `conn Tunnel1` block. This keyword is **not valid** on the libreswan-based `openswan` package installed on Amazon Linux 2, and will throw `keyword auth, invalid value: esp` when the service starts. **Delete that line entirely** if you're copying from the downloaded file — it is intentionally omitted from the config given in Step C5 below.

---

## 7. Lab Part C — Configure Openswan on Singapore EC2

SSH into the **Singapore instance** (`singapore-onprem-gateway`) using its Elastic IP, then run the following **in the corrected order**:

### Step C1 — Switch to root

```bash
sudo su -
```

### Step C2 — Install Openswan

```bash
yum install openswan -y
```

> Only works on **Amazon Linux 2**. If this fails with "no package openswan available," you launched AL2023 by mistake — relaunch with the AL2 AMI (see Edge Cases §2.6).

### Step C3 — Uncomment the include line in ipsec.conf

```bash
vi /etc/ipsec.conf
```

Ensure this line is present and NOT commented out:

```
include /etc/ipsec.d/*.conf
```

### Step C4 — Update kernel networking parameters

```bash
vi /etc/sysctl.conf
```

Add/ensure these three lines exist:

```
net.ipv4.ip_forward = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
```

Apply immediately **without restarting the network service** (restarting the network service on an EC2 instance can drop your SSH session and is unnecessary — `sysctl -p` reloads the file live):

```bash
sysctl -p
```

> **Sequencing correction from your original steps:** `service network restart` is unreliable on EC2 (can hang or drop your active SSH session) and is not needed — `sysctl -p` achieves the same effect safely.

### Step C5 — Create the tunnel config file

```bash
vi /etc/ipsec.d/aws-vpn.conf
```

Paste, filling in the real values from the config file you downloaded in Step B9:

```
conn Tunnel1
        authby=secret
        auto=start
        left=%defaultroute
        leftid=<SINGAPORE_ELASTIC_IP>
        right=<VGW_TUNNEL1_OUTSIDE_IP_FROM_DOWNLOADED_CONFIG>
        type=tunnel
        ikelifetime=8h
        keylife=1h
        phase2alg=aes128-sha1;modp1024
        ike=aes128-sha1;modp1024
        keyingtries=%forever
        keyexchange=ike
        leftsubnet=10.20.0.0/16
        rightsubnet=10.10.0.0/16
        dpddelay=10
        dpdtimeout=30
        dpdaction=restart_by_peer
```

- `leftid` = your Singapore Elastic IP (Customer Gateway public IP)
- `right` = the VGW's Tunnel 1 outside IP address from the downloaded config
- `leftsubnet` = Singapore VPC CIDR (the "on-prem LAN")
- `rightsubnet` = Mumbai VPC CIDR (the "AWS LAN")

### Step C6 — Create the secrets file

```bash
vi /etc/ipsec.d/aws-vpn.secrets
```

```
<SINGAPORE_ELASTIC_IP> <VGW_TUNNEL1_OUTSIDE_IP>: PSK "<PRE_SHARED_KEY_FROM_DOWNLOADED_CONFIG>"
```

Secure the file permissions (Openswan will refuse to start otherwise):

```bash
chmod 600 /etc/ipsec.d/aws-vpn.secrets
```

### Step C7 — Enable and start the ipsec service

```bash
chkconfig ipsec on
service ipsec restart
service ipsec status
```

> **Sequencing correction:** use `restart` (not just `start`) the first time, so any partially-loaded state from the install is cleared. `chkconfig ipsec on` must run before the restart so the service is registered to launch on boot too.

### Step C8 — Verify tunnel state

```bash
ipsec auto --status
```

Look for `Tunnel1` listed as `erouted`/`up` (IPsec SA established). Also check from the **AWS console**:

1. **VPC console → Site-to-Site VPN Connections → mumbai-singapore-vpn → Tunnel Details tab**.
2. Tunnel 1 status should flip from `DOWN` to **`UP`** within a minute of the tunnel negotiating.

---

## 8. Verification / Testing

1. From the **Singapore instance**, ping the Mumbai private IP:
   ```bash
   ping 10.10.1.x
   ```
   (replace `10.10.1.x` with the actual private IP of `mumbai-cloud-workload`, visible in the Mumbai EC2 console)
2. If ping succeeds → tunnel is fully working end-to-end.
3. If ping fails, check in this order:
   - AWS Console Tunnel Details tab — is status UP?
   - `ipsec auto --status` on Singapore instance — is the SA established?
   - Mumbai route table — is VGW propagation actually enabled and showing `10.20.0.0/16 → vgw-xxxx`?
   - Both security groups — ICMP allowed in both directions?
   - Source/Dest check disabled on Singapore instance?

---

## 9. CLI Reference (for later Infrastructure-as-Code practice — not used in the console lab above)

```bash
# Mumbai VPC
aws ec2 create-vpc --cidr-block 10.10.0.0/16 --region ap-south-1

# Singapore VPC
aws ec2 create-vpc --cidr-block 10.20.0.0/16 --region ap-southeast-1

# Customer Gateway (Mumbai region, pointing at Singapore EIP)
aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --public-ip <SINGAPORE_EIP> \
  --bgp-asn 65000 \
  --region ap-south-1

# Virtual Private Gateway (Mumbai)
aws ec2 create-vpn-gateway --type ipsec.1 --region ap-south-1
aws ec2 attach-vpn-gateway --vpn-gateway-id vgw-xxxx --vpc-id vpc-xxxx --region ap-south-1

# VPN Connection (static routing)
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id cgw-xxxx \
  --vpn-gateway-id vgw-xxxx \
  --options StaticRoutesOnly=true \
  --region ap-south-1

# Add static route for Singapore CIDR
aws ec2 create-vpn-connection-route \
  --vpn-connection-id vpn-xxxx \
  --destination-cidr-block 10.20.0.0/16 \
  --region ap-south-1

# Enable route propagation on Mumbai route table
aws ec2 enable-vgw-route-propagation \
  --route-table-id rtb-xxxx \
  --gateway-id vgw-xxxx \
  --region ap-south-1

# Disable source/dest check on Singapore instance
aws ec2 modify-instance-attribute \
  --instance-id i-xxxx \
  --no-source-dest-check \
  --region ap-southeast-1

# Check tunnel status
aws ec2 describe-vpn-connections --vpn-connection-ids vpn-xxxx --region ap-south-1
```

---

## 10. Terraform Reference

```hcl
provider "aws" {
  alias  = "mumbai"
  region = "ap-south-1"
}

provider "aws" {
  alias  = "singapore"
  region = "ap-southeast-1"
}

# --- Singapore: on-prem simulator ---
resource "aws_vpc" "singapore" {
  provider   = aws.singapore
  cidr_block = "10.20.0.0/16"
  tags       = { Name = "singapore-onprem-vpc" }
}

resource "aws_subnet" "singapore_public" {
  provider                = aws.singapore
  vpc_id                  = aws_vpc.singapore.id
  cidr_block              = "10.20.1.0/24"
  availability_zone       = "ap-southeast-1a"
  map_public_ip_on_launch = true
  tags                    = { Name = "singapore-public-subnet" }
}

resource "aws_internet_gateway" "singapore" {
  provider = aws.singapore
  vpc_id   = aws_vpc.singapore.id
  tags     = { Name = "singapore-igw" }
}

resource "aws_route_table" "singapore" {
  provider = aws.singapore
  vpc_id   = aws_vpc.singapore.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.singapore.id
  }
  tags = { Name = "singapore-public-rt" }
}

resource "aws_route_table_association" "singapore" {
  provider       = aws.singapore
  subnet_id      = aws_subnet.singapore_public.id
  route_table_id = aws_route_table.singapore.id
}

resource "aws_security_group" "singapore_vpn" {
  provider = aws.singapore
  vpc_id   = aws_vpc.singapore.id
  name     = "singapore-vpn-sg"

  ingress { from_port = 22   to_port = 22   protocol = "tcp" cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 500  to_port = 500  protocol = "udp" cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 4500 to_port = 4500 protocol = "udp" cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 0    to_port = 0    protocol = "50"  cidr_blocks = ["0.0.0.0/0"] } # ESP
  ingress { from_port = -1   to_port = -1   protocol = "icmp" cidr_blocks = ["10.10.0.0/16"] }
  egress  { from_port = 0    to_port = 0    protocol = "-1"  cidr_blocks = ["0.0.0.0/0"] }
}

resource "aws_instance" "singapore_gateway" {
  provider                   = aws.singapore
  ami                        = "ami-0xxxxxxxxxxxxxxxx" # Amazon Linux 2 AMI in ap-southeast-1
  instance_type               = "t2.micro"
  subnet_id                   = aws_subnet.singapore_public.id
  vpc_security_group_ids      = [aws_security_group.singapore_vpn.id]
  source_dest_check           = false
  tags                        = { Name = "singapore-onprem-gateway" }
}

resource "aws_eip" "singapore" {
  provider = aws.singapore
  instance = aws_instance.singapore_gateway.id
  domain   = "vpc"
}

# --- Mumbai: AWS cloud side ---
resource "aws_vpc" "mumbai" {
  provider   = aws.mumbai
  cidr_block = "10.10.0.0/16"
  tags       = { Name = "mumbai-cloud-vpc" }
}

resource "aws_subnet" "mumbai_private" {
  provider          = aws.mumbai
  vpc_id            = aws_vpc.mumbai.id
  cidr_block        = "10.10.1.0/24"
  availability_zone = "ap-south-1a"
  tags              = { Name = "mumbai-private-subnet" }
}

resource "aws_route_table" "mumbai" {
  provider = aws.mumbai
  vpc_id   = aws_vpc.mumbai.id
  tags     = { Name = "mumbai-vpn-rt" }
}

resource "aws_route_table_association" "mumbai" {
  provider       = aws.mumbai
  subnet_id      = aws_subnet.mumbai_private.id
  route_table_id = aws_route_table.mumbai.id
}

resource "aws_security_group" "mumbai_vpn" {
  provider = aws.mumbai
  vpc_id   = aws_vpc.mumbai.id
  name     = "mumbai-vpn-sg"

  ingress { from_port = -1 to_port = -1 protocol = "icmp" cidr_blocks = ["10.20.0.0/16"] }
  ingress { from_port = 22 to_port = 22 protocol = "tcp"  cidr_blocks = ["10.20.0.0/16"] }
  egress  { from_port = 0  to_port = 0  protocol = "-1"   cidr_blocks = ["0.0.0.0/0"] }
}

resource "aws_instance" "mumbai_workload" {
  provider               = aws.mumbai
  ami                     = "ami-0xxxxxxxxxxxxxxxx" # Amazon Linux 2 AMI in ap-south-1
  instance_type           = "t2.micro"
  subnet_id               = aws_subnet.mumbai_private.id
  vpc_security_group_ids  = [aws_security_group.mumbai_vpn.id]
  tags                    = { Name = "mumbai-cloud-workload" }
}

resource "aws_customer_gateway" "singapore" {
  provider   = aws.mumbai
  bgp_asn    = 65000
  ip_address = aws_eip.singapore.public_ip
  type       = "ipsec.1"
  tags       = { Name = "singapore-cgw" }
}

resource "aws_vpn_gateway" "mumbai" {
  provider = aws.mumbai
  vpc_id   = aws_vpc.mumbai.id
  tags     = { Name = "mumbai-vgw" }
}

resource "aws_vpn_gateway_route_propagation" "mumbai" {
  provider       = aws.mumbai
  vpn_gateway_id = aws_vpn_gateway.mumbai.id
  route_table_id = aws_route_table.mumbai.id
}

resource "aws_vpn_connection" "main" {
  provider            = aws.mumbai
  customer_gateway_id = aws_customer_gateway.singapore.id
  vpn_gateway_id      = aws_vpn_gateway.mumbai.id
  type                = "ipsec.1"
  static_routes_only  = true
  tags                = { Name = "mumbai-singapore-vpn" }
}

resource "aws_vpn_connection_route" "singapore_cidr" {
  provider                = aws.mumbai
  vpn_connection_id       = aws_vpn_connection.main.id
  destination_cidr_block  = "10.20.0.0/16"
}
```

> Terraform provisions the AWS-side resources and the EC2 instance itself but **cannot configure Openswan inside the OS** — Part C (SSH + manual Openswan config, or a `user_data` bootstrap script referencing the PSK output from `aws_vpn_connection.main.tunnel1_preshared_key`) is still a manual/scripted post-step.

---

## 11. Troubleshooting / Edge Cases Checklist

| Symptom | Likely Cause | Fix |
|---|---|---|
| `yum install openswan` fails | Instance is AL2023, not AL2 | Relaunch with Amazon Linux 2 AMI |
| Tunnel shows DOWN in console indefinitely | SG blocking UDP 500/4500/ESP | Open those ports/protocol inbound on Singapore SG |
| `ipsec auto --status` shows no SA | PSK mismatch between `.secrets` file and downloaded config | Re-copy PSK exactly, no extra whitespace |
| Tunnel UP but ping fails | Source/Dest check still enabled on Singapore instance | Disable it (Step A8) |
| Tunnel UP, ping fails, SG looks fine | Mumbai route table missing VGW propagation | Enable VGW route propagation on `mumbai-vpn-rt` |
| Public IP of Singapore instance changed | Instance stopped/started without EIP | Always use Elastic IP, not the ephemeral public IP |
| SSH session drops during config | Ran `service network restart` | Use `sysctl -p` instead, avoid restarting network service on EC2 |
| Tunnel flaps up/down repeatedly | DPD (dead peer detection) mismatch or NAT device between endpoints re-mapping ports | Check `dpddelay`/`dpdtimeout`, ensure NAT-T (UDP 4500) is allowed |
| `ipsec restart` fails with `keyword auth, invalid value: esp` | Leftover `auth=esp` line copied from the downloaded Openswan config | Delete the `auth=esp` line — it's not a valid keyword on AL2's libreswan-based openswan package |
| `esp="aes128-sha1;modp1024" is invalid: ESP encryption algorithm 'aes' is not supported` | Algorithm string format rejected by the installed openswan/libreswan version | Try `aes128-sha1-modp1024` (hyphens instead of semicolon) as an alternate `phase2alg`/`ike` syntax, or check `ipsec --version` for the exact algorithm-naming convention it expects |

---

## 12. Interview Q&A

**Q1: What's the difference between Site-to-Site VPN and Direct Connect?**
A: VPN rides the public internet with IPSec encryption, provisions in minutes, capped around 1.25 Gbps per tunnel. Direct Connect is a dedicated physical circuit into an AWS Direct Connect location, offers consistent low latency and higher bandwidth (up to 100 Gbps), but takes weeks to provision and costs more. VPN is often used as a Direct Connect failover.

**Q2: Why does an AWS VPN connection always have two tunnels?**
A: For high availability. Each tunnel terminates on a different AWS public IP in a different underlying device/facility, so if AWS performs maintenance on one endpoint, the second tunnel keeps traffic flowing without downtime.

**Q3: What is a Customer Gateway in AWS terms?**
A: It's a **logical AWS resource/object** representing your on-premises (or in this lab, "on-prem simulated") VPN device — it stores that device's public IP and BGP ASN. It is NOT the physical device itself; the actual device (in this lab, the Singapore EC2 running Openswan) is what terminates the tunnel.

**Q4: What's the difference between static routing and BGP (dynamic) routing for a VPN connection?**
A: Static routing means you manually declare which CIDR ranges are reachable behind the Customer Gateway. BGP dynamic routing means the on-prem router and AWS VGW exchange routes automatically over the tunnel — more resilient to CIDR changes but requires a BGP-capable device and more configuration.

**Q5: Why must Source/Destination Check be disabled on the instance acting as the customer gateway?**
A: Because that EC2 instance is forwarding/routing traffic that isn't natively addressed to or from itself (it's encapsulating other hosts' traffic inside IPSec). AWS's default check drops any packet where the instance isn't the literal source or destination, which would silently break the tunnel.

**Q6: What protocols/ports must be open for an IPSec Site-to-Site VPN to establish?**
A: UDP 500 (IKE — key exchange), UDP 4500 (IPSec NAT-Traversal), and IP protocol 50 (ESP — Encapsulating Security Payload, the actual encrypted data).

**Q7: Can Tunnel1 and Tunnel2 both be active at the same time on a plain VGW-based VPN connection?**
A: Not for active-active load balancing on a lone VGW — Tunnel2 is a standby/failover path. True active-active (ECMP) across multiple tunnels requires attaching the VPN to a **Transit Gateway** with ECMP support enabled.

**Q8: What's the maximum throughput of a single AWS VPN tunnel?**
A: Approximately 1.25 Gbps per tunnel — this is an AWS-imposed limit, not something you can raise by resizing instances.

**Q9: Is `yum install openswan` on Amazon Linux 2 actually installing Openswan?**
A: Not literally. The original Openswan project is unmaintained, so on Amazon Linux 2 the `openswan` package name resolves as a compatibility alias that pulls in **Libreswan** (a maintained fork) under the hood. The config file locations, `ipsec` command, and file formats stay the same, which is why Openswan-style tutorials still work on AL2 — but it explains why some strictly-Openswan-only keywords (like `auth=esp`) are rejected: the underlying binary is Libreswan, which dropped that keyword. On Amazon Linux 2023, there's no alias at all — you must add a Fedora/EPEL-style repo and install `libreswan` directly.

---

## 13. Cheat Sheet

```
Mumbai (AWS side)        = VPC 10.10.0.0/16, private EC2, VGW, CGW object, VPN connection, route propagation
Singapore (on-prem side) = VPC 10.20.0.0/16, public EC2 + EIP, IGW, Openswan tunnel termination

Order of creation:
  1. Singapore VPC → Subnet → IGW → RouteTable → SG → EC2 → EIP → disable src/dst check
  2. Mumbai VPC → Subnet → RouteTable → SG → EC2
  3. Mumbai: Customer Gateway (uses Singapore EIP)
  4. Mumbai: Virtual Private Gateway → attach to VPC → enable route propagation
  5. Mumbai: VPN Connection (static routing, destination = Singapore CIDR)
  6. Mumbai: Download configuration (vendor = Openswan, or Generic as fallback)
  7. Singapore: install + configure Openswan using downloaded values
  8. Verify: console Tunnel Details tab = UP, then ping across

Key files on Singapore instance:
  /etc/ipsec.conf                → uncomment include line
  /etc/sysctl.conf               → ip_forward=1, accept_redirects=0, send_redirects=0 (apply via sysctl -p)
  /etc/ipsec.d/aws-vpn.conf      → tunnel definition (left=EIP, right=VGW IP, subnets)
  /etc/ipsec.d/aws-vpn.secrets   → PSK (chmod 600)

Service commands:
  chkconfig ipsec on
  service ipsec restart
  service ipsec status
  ipsec auto --status
```

---

## 14. Mastery Checklist

- [ ] Explained what Site-to-Site VPN is and why/when to use it vs Direct Connect
- [ ] Built Singapore VPC + subnet + IGW + route table + SG + EC2 + EIP
- [ ] Disabled Source/Destination check on the Singapore instance
- [ ] Built Mumbai VPC + subnet + route table + SG + private EC2
- [ ] Created Customer Gateway in Mumbai pointing at Singapore's EIP
- [ ] Created Virtual Private Gateway, attached it, enabled route propagation
- [ ] Created VPN Connection with static routing to Singapore's CIDR
- [ ] Downloaded Openswan-vendor configuration and extracted PSK + endpoint IPs (and removed the `auth=esp` line if present)
- [ ] Installed and configured Openswan on the Singapore instance in correct sequence
- [ ] Verified Tunnel1 status = UP in the AWS console
- [ ] Successfully pinged Mumbai private IP from Singapore instance
- [ ] Can explain the difference between Tunnel1/Tunnel2 redundancy and true active-active ECMP
- [ ] Cleaned up: deleted VPN connection, VGW detach/delete, CGW delete, terminated both EC2 instances, released EIP
