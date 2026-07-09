# AWS VPC Peering — Mumbai ↔ Singapore Hands-On Lab Guide

---

## 1. Scenario

You have two separate VPCs — one in **Mumbai (ap-south-1)** and one in **Singapore (ap-southeast-1)** — each with its own subnet, EC2 instance, and security group. They currently have no way to talk to each other privately: any traffic between them would have to leave AWS and go over the public internet (assuming both had public IPs), which is slow, insecure, and unnecessary since both are already inside AWS.

**VPC Peering** solves this: it's a direct, non-transitive private network connection between two VPCs (same account or cross-account, same region or cross-region) that lets resources in either VPC talk to each other using **private IP addresses**, as if they were on the same network — with no gateway, no VPN, no extra appliance, and (in most cases) no extra hourly charge.

Goal: create a VPC Peering connection between Mumbai and Singapore, accept it, update both route tables and security groups, and verify that an EC2 instance in one region can ping/SSH an EC2 instance in the other region using only its **private IP** — proving traffic never leaves the AWS backbone.

This lab is deliberately built **cross-region** (rather than same-region) because it surfaces every real interview-relevant edge case: manual VPC-ID entry across regions, inter-region MTU limits, and the security-group-referencing restriction that only applies across regions.

---

## 2. Concept Deep Dive

### 2.1 What is VPC Peering?

A VPC Peering connection is a **networking connection between two VPCs** that enables routing traffic between them using private IPv4 (or IPv6) addresses. AWS implements it using the existing infrastructure of a VPC — it is **not** a gateway, not a VPN connection, and does not rely on a separate physical appliance or a single point of failure. Instances in either VPC communicate with each other as if they were in the same network.

Peering connections can be:
- **Intra-region** (both VPCs in the same AWS Region)
- **Inter-region** (VPCs in different Regions — used in this lab)
- **Intra-account** (both VPCs owned by the same account — used in this lab)
- **Cross-account** (VPCs owned by different AWS accounts)

### 2.2 Core Components

| Component | What it is | Where it lives in this lab |
|---|---|---|
| **Requester VPC** | The VPC that initiates the peering connection request. | `mumbai-peer-vpc` (`ap-south-1`) |
| **Accepter VPC** | The VPC whose owner must explicitly accept the request before traffic can flow. | `singapore-peer-vpc` (`ap-southeast-1`) |
| **VPC Peering Connection (pcx-xxxx)** | The AWS resource representing the relationship itself. Goes through states: `pending-acceptance` → `active` (or `rejected`/`expired`/`deleted`/`failed`). | `mumbai-singapore-pcx` |
| **Route Table entries** | Peering does **not** auto-propagate routes (unlike a VGW attachment). You must manually add a route in each VPC's route table pointing the peer's CIDR at the peering connection ID. | Added to both `mumbai-peer-rt` and `singapore-peer-rt` |
| **Security Group rules** | Must explicitly allow the desired traffic (SSH, ICMP, etc.) from the peer VPC's CIDR. For inter-region peering, you **cannot** reference the peer's security group ID directly — only CIDR blocks are allowed across regions. | CIDR-based rules on both sides |
| **DNS Resolution Support** | An optional setting per peering connection that lets a public DNS hostname of an instance in the peer VPC resolve to its **private** IP instead of its public IP, keeping traffic off the internet even when addressed by hostname. Supported for both same-region and inter-region peering. | Enabled optionally in Lab Part F |

### 2.3 Why use VPC Peering?

- Let multiple teams/environments (e.g., a shared-services VPC, a data VPC, an app VPC) communicate privately without merging them into one giant VPC.
- Connect VPCs across regions for **multi-region architectures**, disaster recovery replication, or serving users from the nearest region while sharing a common backend.
- Avoid routing inter-VPC traffic over the public internet, reducing exposure to internet-based threats and avoiding NAT Gateway data-processing charges for that traffic.
- Cheaper and simpler than Transit Gateway for a **small number** of VPCs that need to talk to each other directly.
- No bandwitdth bottleneck or single point of failure, since it rides the same redundant infrastructure that powers the VPC itself.

### 2.4 Benefits

| Benefit | Detail |
|---|---|
| No extra appliance | Not a gateway or VPN — uses the VPC's own infrastructure |
| No peering-connection hourly charge | Free to create; you only pay standard data transfer rates for traffic that crosses an AZ or Region |
| Works across accounts and regions | Same feature covers same-account/same-region up to cross-account/cross-region |
| Full bandwidth | No inherent bandwidth cap between peered instances beyond normal EC2/network limits |
| Encrypted in transit for inter-region | Cross-region peering traffic is automatically encrypted using AEAD algorithms on the AWS backbone — no configuration needed |
| Traffic never touches the public internet | Reduces exposure to common internet-based exploits and DDoS |

### 2.5 Limitations

- **Non-transitive**: if VPC A peers with VPC B, and VPC A also peers with VPC C, resources in B **cannot** reach C through A. A direct B↔C peering connection would be required.
- **No edge-to-edge routing through a peer's gateway**: if VPC A has an Internet Gateway, NAT Gateway, VPN connection, or Direct Connect connection, VPC B **cannot** use any of those through the peering connection to reach the internet or on-prem — each VPC must have its own path out.
- **CIDRs must not overlap**: a peering connection cannot be created (and won't be usable) between VPCs with matching or overlapping CIDR blocks.
- **Security group referencing is region-scoped**: you can reference a peer VPC's security group ID directly as a source **only if both VPCs are in the same Region**. For inter-region peering (this lab), you must use CIDR blocks in your security group rules instead.
- **No Jumbo Frames across regions**: intra-region peering supports up to 9001-byte MTU (jumbo frames); inter-region peering is capped at **1500-byte MTU**.
- **IPv6 is not supported for inter-region peering** (only IPv4). Same-region peering can support IPv6 if both VPCs have it enabled.
- **Quotas**: default limit of 50 active VPC peering connections per VPC (can be raised via support request up to 125); only one peering connection allowed between the same two VPCs at a time; unaccepted requests automatically expire after **168 hours (7 days)**.
- **No DNS resolution or route propagation by default** — both must be explicitly configured; nothing is automatic the way VGW route propagation is.

### 2.6 Cost

- **Creating a VPC Peering connection itself is free** — no hourly charge for the connection resource, unlike a Site-to-Site VPN or Client VPN endpoint.
- **Data transfer within the same Availability Zone** over a peering connection is free, even across different AWS accounts.
- **Data transfer that crosses an Availability Zone** (same-region peering, different AZ) or **crosses a Region** (inter-region peering, this lab) is billed at standard AWS data transfer rates — inter-region data transfer is typically the more expensive tier, so treat cross-region peering as a deliberate architectural choice, not a default.
- EC2 instance costs for the two lab instances (t2.micro/t3.micro, free-tier eligible in most accounts) are separate and apply regardless of peering.
- **Remember to delete the peering connection and terminate both EC2 instances after the lab** — while the peering connection itself has no hourly charge, the EC2 instances do.

### 2.7 Edge Cases

- **Cross-region VPC selection isn't a dropdown.** When creating the peering connection from Mumbai, the console cannot browse Singapore's VPCs for you — you must manually paste Singapore's VPC ID (copied from the Singapore console beforehand).
- **Overlapping CIDRs silently break the lab.** This lab uses `10.50.0.0/16` (Mumbai) and `10.60.0.0/16` (Singapore) specifically because they don't overlap — always check this before starting, since AWS will reject the peering connection creation if VPCs are in the same account/region... but if they're in different accounts, an overlap may not be rejected outright at creation time, so verify manually regardless.
- **Route tables need entries on BOTH sides**, and there's no propagation feature like a VGW attachment has — each VPC's route table needs a manually added static route pointing the peer's CIDR at the `pcx-xxxx` ID.
- **Security groups need entries on BOTH sides too** — and because this is inter-region, you must use the peer's CIDR block, not a security-group-ID reference (SG-to-SG referencing across regions isn't supported by AWS).
- **"Active" doesn't guarantee reachability.** A peering connection can show `Active` in the console even if a regional event is preventing traffic flow, or if route tables/security groups aren't correctly configured — `Active` only confirms the peering relationship exists, not that traffic is actually flowing.
- **DNS resolution must be enabled on both sides independently** if you want hostname-based resolution to work bidirectionally — enabling it only on the requester's side means only the requester benefits.
- **Unaccepted requests expire in 7 days.** If you create the peering connection and forget to accept it from the Singapore console, it will silently expire after 168 hours and you'll need to recreate it.

### 2.8 Assumptions

- You have an AWS account with permissions to create VPCs, EC2 instances, security groups, route tables, and VPC Peering connections in both `ap-south-1` and `ap-southeast-1`.
- Both VPCs are owned by the **same AWS account** in this lab (intra-account, inter-region) — cross-account steps are noted where they'd differ, but not performed here.
- CIDR plan (deliberately non-overlapping):
  - Mumbai VPC (Requester): `10.50.0.0/16` → subnet `10.50.1.0/24`
  - Singapore VPC (Accepter): `10.60.0.0/16` → subnet `10.60.1.0/24`
- Both subnets are **public** (have an Internet Gateway) purely so you can SSH into each instance independently for setup/troubleshooting — the actual peering test itself uses **private IPs only**, proving no internet path is involved for inter-VPC traffic.

---

## 3. Architecture Diagram

```
        MUMBAI (ap-south-1)                          SINGAPORE (ap-southeast-1)
        VPC 10.50.0.0/16                              VPC 10.60.0.0/16
        ┌─────────────────────────┐                   ┌─────────────────────────┐
        │  Subnet 10.50.1.0/24     │                   │  Subnet 10.60.1.0/24     │
        │  ┌───────────────────┐  │                   │  ┌───────────────────┐  │
        │  │  EC2 Instance A    │  │                   │  │  EC2 Instance B    │  │
        │  │  10.50.1.x         │  │                   │  │  10.60.1.x         │  │
        │  │  (public IP too,   │  │                   │  │  (public IP too,   │  │
        │  │   for admin SSH)   │  │                   │  │   for admin SSH)   │  │
        │  └─────────┬──────────┘  │                   │  └─────────┬──────────┘  │
        │            │             │                   │            │             │
        │      Route Table         │                   │      Route Table         │
        │  10.50.0.0/16 → local    │                   │  10.60.0.0/16 → local    │
        │  10.60.0.0/16 → pcx-xxxx │◄──────────────────►│  10.50.0.0/16 → pcx-xxxx │
        │  0.0.0.0/0    → IGW      │                   │  0.0.0.0/0    → IGW      │
        │            │             │   VPC Peering     │            │             │
        │      Internet Gateway    │   Connection       │      Internet Gateway    │
        └─────────────────────────┘   (pcx-xxxx)        └─────────────────────────┘
                                    Non-transitive,
                                 no gateway, no VPN —
                              uses AWS's own backbone
```

---

## 4. Prerequisites

- IAM permissions: `ec2:*` for VPC, Subnet, EC2, Security Group, Route Table, and VPC Peering Connection operations in both `ap-south-1` and `ap-southeast-1`.
- A key pair created in **each** region (or reuse if your workflow allows cross-region key import) to SSH into both EC2 instances directly for setup/testing.
- A terminal / SSH client.

---

## 5. Lab Part A — Mumbai VPC (Requester)

### Step A1 — Create VPC

1. Console region: **Asia Pacific (Mumbai) ap-south-1**.
2. **VPC console → Your VPCs → Create VPC**.
3. Select **VPC only**.
4. Name tag: `mumbai-peer-vpc`
5. IPv4 CIDR: `10.50.0.0/16`
6. Create VPC.

### Step A2 — Create Subnet

1. **VPC console → Subnets → Create subnet**.
2. VPC: `mumbai-peer-vpc`
3. Name: `mumbai-peer-subnet`
4. AZ: `ap-south-1a`
5. IPv4 CIDR: `10.50.1.0/24`
6. Create subnet.
7. Select it → **Actions → Edit subnet settings** → enable **Auto-assign public IPv4 address** → Save.

### Step A3 — Create and Attach Internet Gateway

1. **VPC console → Internet Gateways → Create internet gateway**.
2. Name: `mumbai-peer-igw`
3. Create, then select it → **Actions → Attach to VPC** → choose `mumbai-peer-vpc`.

### Step A4 — Create Route Table

1. **VPC console → Route Tables → Create route table**.
2. Name: `mumbai-peer-rt`
3. VPC: `mumbai-peer-vpc`
4. Create.
5. **Routes tab → Edit routes → Add route** → Destination: `0.0.0.0/0` → Target: `mumbai-peer-igw` → Save.
6. **Subnet associations tab → Edit subnet associations** → select `mumbai-peer-subnet` → Save.

### Step A5 — Create Security Group

1. **EC2 console → Security Groups → Create security group**.
2. Name: `mumbai-peer-sg`
3. VPC: `mumbai-peer-vpc`
4. Inbound rules (for now):
   - SSH (22) — Source: My IP
5. Create. *(You'll add a rule for Singapore's CIDR after peering is set up, in Lab Part E.)*

### Step A6 — Launch EC2 Instance

1. **EC2 console → Launch instance**.
2. Name: `mumbai-peer-instance`
3. AMI: Amazon Linux 2.
4. Instance type: `t2.micro`.
5. Key pair: select/create one for Mumbai.
6. Network settings: VPC `mumbai-peer-vpc`, subnet `mumbai-peer-subnet`, auto-assign public IP enabled, security group `mumbai-peer-sg`.
7. Launch. **Note both its private IP (`10.50.1.x`) and public IP.**

---

## 6. Lab Part B — Singapore VPC (Accepter)

### Step B1 — Create VPC

1. Switch console region to **Asia Pacific (Singapore) ap-southeast-1**.
2. **VPC console → Your VPCs → Create VPC**.
3. Select **VPC only**.
4. Name tag: `singapore-peer-vpc`
5. IPv4 CIDR: `10.60.0.0/16`
6. Create VPC. **Note the VPC ID (`vpc-xxxx`) — you'll need to paste this manually in Mumbai when creating the peering connection.**

### Step B2 — Create Subnet

1. **VPC console → Subnets → Create subnet**.
2. VPC: `singapore-peer-vpc`
3. Name: `singapore-peer-subnet`
4. AZ: `ap-southeast-1a`
5. IPv4 CIDR: `10.60.1.0/24`
6. Create subnet.
7. Select it → **Actions → Edit subnet settings** → enable **Auto-assign public IPv4 address** → Save.

### Step B3 — Create and Attach Internet Gateway

1. **VPC console → Internet Gateways → Create internet gateway**.
2. Name: `singapore-peer-igw`
3. Create, then select it → **Actions → Attach to VPC** → choose `singapore-peer-vpc`.

### Step B4 — Create Route Table

1. **VPC console → Route Tables → Create route table**.
2. Name: `singapore-peer-rt`
3. VPC: `singapore-peer-vpc`
4. Create.
5. **Routes tab → Edit routes → Add route** → Destination: `0.0.0.0/0` → Target: `singapore-peer-igw` → Save.
6. **Subnet associations tab → Edit subnet associations** → select `singapore-peer-subnet` → Save.

### Step B5 — Create Security Group

1. **EC2 console → Security Groups → Create security group**.
2. Name: `singapore-peer-sg`
3. VPC: `singapore-peer-vpc`
4. Inbound rules (for now):
   - SSH (22) — Source: My IP
5. Create. *(You'll add a rule for Mumbai's CIDR after peering is set up, in Lab Part E.)*

### Step B6 — Launch EC2 Instance

1. **EC2 console → Launch instance**.
2. Name: `singapore-peer-instance`
3. AMI: Amazon Linux 2.
4. Instance type: `t2.micro`.
5. Key pair: select/create one for Singapore.
6. Network settings: VPC `singapore-peer-vpc`, subnet `singapore-peer-subnet`, auto-assign public IP enabled, security group `singapore-peer-sg`.
7. Launch. **Note both its private IP (`10.60.1.x`) and public IP.**

---

## 7. Lab Part C — Create and Accept the Peering Connection

### Step C1 — Create the peering connection (from Mumbai, the Requester)

1. Switch console region back to **Mumbai (ap-south-1)**.
2. **VPC console → Peering Connections → Create Peering Connection**.
3. Peering connection name tag: `mumbai-singapore-pcx`
4. VPC (Requester): select `mumbai-peer-vpc` (this is filled automatically since you're in this region/VPC context).
5. Select another account: leave as **My account** (same-account lab).
6. Region: select **Another Region** → choose `ap-southeast-1` (Singapore).
7. VPC (Accepter): paste the **Singapore VPC ID** you noted in Step B1 — the console cannot browse it for you since it's in a different Region.
8. Create Peering Connection.
9. Status will show `Initiating-request`, then `Pending-acceptance`.

### Step C2 — Accept the peering connection (from Singapore, the Accepter)

1. Switch console region to **Singapore (ap-southeast-1)**.
2. **VPC console → Peering Connections**.
3. You should see `mumbai-singapore-pcx` with status `Pending Acceptance`.
4. Select it → **Actions → Accept Request**.
5. Confirm. Status changes to `Active`.

> **Edge case reminder:** if you don't accept within 7 days (168 hours), the request expires automatically and you'll have to recreate it from Mumbai.

---

## 8. Lab Part D — Update Route Tables (Both Sides)

Peering does **not** auto-propagate routes — you must add a static route on each side manually.

### Step D1 — Mumbai route table

1. Switch to **Mumbai (ap-south-1)**.
2. **VPC console → Route Tables → `mumbai-peer-rt` → Routes tab → Edit routes → Add route**.
3. Destination: `10.60.0.0/16` (Singapore's VPC CIDR)
4. Target: **Peering Connection** → select `mumbai-singapore-pcx`
5. Save routes.

### Step D2 — Singapore route table

1. Switch to **Singapore (ap-southeast-1)**.
2. **VPC console → Route Tables → `singapore-peer-rt` → Routes tab → Edit routes → Add route**.
3. Destination: `10.50.0.0/16` (Mumbai's VPC CIDR)
4. Target: **Peering Connection** → select `mumbai-singapore-pcx`
5. Save routes.

---

## 9. Lab Part E — Update Security Groups (Both Sides)

### Step E1 — Mumbai security group

1. Switch to **Mumbai (ap-south-1)**.
2. **EC2 console → Security Groups → `mumbai-peer-sg` → Inbound rules → Edit inbound rules → Add rule**.
3. Type: SSH (22) — Source: `10.60.0.0/16` (Singapore's VPC CIDR)
4. Add another rule: Type: All ICMP – IPv4 — Source: `10.60.0.0/16`
5. Save rules.

### Step E2 — Singapore security group

1. Switch to **Singapore (ap-southeast-1)**.
2. **EC2 console → Security Groups → `singapore-peer-sg` → Inbound rules → Edit inbound rules → Add rule**.
3. Type: SSH (22) — Source: `10.50.0.0/16` (Mumbai's VPC CIDR)
4. Add another rule: Type: All ICMP – IPv4 — Source: `10.50.0.0/16`
5. Save rules.

> **Reminder:** because this is an **inter-region** peering connection, you must use CIDR blocks here — you cannot select "the Singapore security group" as a source directly the way you could if both VPCs were in the same Region.

---

## 10. Lab Part F — (Optional) Enable DNS Resolution Support

This lets a public DNS hostname of an instance in one VPC resolve to its **private** IP address when queried from the peer VPC, instead of resolving to its public IP.

1. From either region's console → **VPC console → Peering Connections → `mumbai-singapore-pcx`**.
2. **Actions → Edit DNS Settings**.
3. Check **Enable DNS resolution for queries from the peer VPC** (this option exists independently on both the requester's and accepter's side).
4. Save.
5. Repeat from the other region's console to enable it on both sides if you want bidirectional hostname resolution.

> This is supported for both same-region and inter-region peering connections (AWS added inter-region DNS resolution support some years ago) — it is not required for the ping/SSH-by-private-IP test in the next section, so feel free to skip it if you only want to prove basic connectivity.

---

## 11. Verification / Testing

1. SSH into the **Mumbai instance** using its public IP (via your key pair).
2. From inside the Mumbai instance, ping the Singapore instance's **private IP**:
   ```bash
   ping 10.60.1.x
   ```
3. Also try SSH from Mumbai to Singapore using the private IP (requires the Singapore key pair to be available on the Mumbai instance, or copy it over):
   ```bash
   ssh -i singapore-key.pem ec2-user@10.60.1.x
   ```
4. If both succeed → the peering connection, route tables, and security groups are all correctly configured.
5. If ping/SSH fails, check in this order:
   - **Peering Connection status** — is it `Active` (not still `Pending-acceptance`)?
   - **Both route tables** — does each have a route for the *other* VPC's CIDR pointing at `pcx-xxxx`?
   - **Both security groups** — does each allow the *other* VPC's CIDR inbound for the protocol you're testing?
   - Confirm you're testing with **private IPs**, not public IPs — pinging the public IP would go over the internet, not through the peering connection, and defeats the purpose of the test.

---

## 12. CLI Reference (for later Infrastructure-as-Code practice — not used in the console lab above)

```bash
# Create VPCs
aws ec2 create-vpc --cidr-block 10.50.0.0/16 --region ap-south-1
aws ec2 create-vpc --cidr-block 10.60.0.0/16 --region ap-southeast-1

# Create the peering connection (run from Mumbai / requester region)
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-mumbai-xxxx \
  --peer-vpc-id vpc-singapore-xxxx \
  --peer-region ap-southeast-1 \
  --region ap-south-1

# Accept the peering connection (run from Singapore / accepter region)
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id pcx-xxxx \
  --region ap-southeast-1

# Add route in Mumbai route table
aws ec2 create-route \
  --route-table-id rtb-mumbai-xxxx \
  --destination-cidr-block 10.60.0.0/16 \
  --vpc-peering-connection-id pcx-xxxx \
  --region ap-south-1

# Add route in Singapore route table
aws ec2 create-route \
  --route-table-id rtb-singapore-xxxx \
  --destination-cidr-block 10.50.0.0/16 \
  --vpc-peering-connection-id pcx-xxxx \
  --region ap-southeast-1

# Add security group rules (CIDR-based, required for inter-region)
aws ec2 authorize-security-group-ingress \
  --group-id sg-mumbai-xxxx \
  --protocol tcp --port 22 --cidr 10.60.0.0/16 \
  --region ap-south-1

aws ec2 authorize-security-group-ingress \
  --group-id sg-singapore-xxxx \
  --protocol tcp --port 22 --cidr 10.50.0.0/16 \
  --region ap-southeast-1

# Enable DNS resolution support on both sides (optional)
aws ec2 modify-vpc-peering-connection-options \
  --vpc-peering-connection-id pcx-xxxx \
  --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=true \
  --region ap-south-1

aws ec2 modify-vpc-peering-connection-options \
  --vpc-peering-connection-id pcx-xxxx \
  --accepter-peering-connection-options AllowDnsResolutionFromRemoteVpc=true \
  --region ap-southeast-1

# Check status
aws ec2 describe-vpc-peering-connections --vpc-peering-connection-ids pcx-xxxx --region ap-south-1
```

---

## 13. Terraform Reference

```hcl
provider "aws" {
  alias  = "mumbai"
  region = "ap-south-1"
}

provider "aws" {
  alias  = "singapore"
  region = "ap-southeast-1"
}

# --- Mumbai VPC ---
resource "aws_vpc" "mumbai" {
  provider   = aws.mumbai
  cidr_block = "10.50.0.0/16"
  tags       = { Name = "mumbai-peer-vpc" }
}

resource "aws_subnet" "mumbai" {
  provider                = aws.mumbai
  vpc_id                  = aws_vpc.mumbai.id
  cidr_block              = "10.50.1.0/24"
  availability_zone       = "ap-south-1a"
  map_public_ip_on_launch = true
  tags                    = { Name = "mumbai-peer-subnet" }
}

resource "aws_internet_gateway" "mumbai" {
  provider = aws.mumbai
  vpc_id   = aws_vpc.mumbai.id
  tags     = { Name = "mumbai-peer-igw" }
}

resource "aws_route_table" "mumbai" {
  provider = aws.mumbai
  vpc_id   = aws_vpc.mumbai.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.mumbai.id
  }
  tags = { Name = "mumbai-peer-rt" }
}

resource "aws_route_table_association" "mumbai" {
  provider       = aws.mumbai
  subnet_id      = aws_subnet.mumbai.id
  route_table_id = aws_route_table.mumbai.id
}

resource "aws_security_group" "mumbai" {
  provider = aws.mumbai
  vpc_id   = aws_vpc.mumbai.id
  name     = "mumbai-peer-sg"

  ingress { from_port = 22 to_port = 22 protocol = "tcp"  cidr_blocks = ["10.60.0.0/16"] }
  ingress { from_port = -1 to_port = -1 protocol = "icmp" cidr_blocks = ["10.60.0.0/16"] }
  egress  { from_port = 0  to_port = 0  protocol = "-1"   cidr_blocks = ["0.0.0.0/0"] }
}

resource "aws_instance" "mumbai" {
  provider                = aws.mumbai
  ami                      = "ami-0xxxxxxxxxxxxxxxx" # Amazon Linux 2 in ap-south-1
  instance_type            = "t2.micro"
  subnet_id                = aws_subnet.mumbai.id
  vpc_security_group_ids   = [aws_security_group.mumbai.id]
  tags                     = { Name = "mumbai-peer-instance" }
}

# --- Singapore VPC ---
resource "aws_vpc" "singapore" {
  provider   = aws.singapore
  cidr_block = "10.60.0.0/16"
  tags       = { Name = "singapore-peer-vpc" }
}

resource "aws_subnet" "singapore" {
  provider                = aws.singapore
  vpc_id                  = aws_vpc.singapore.id
  cidr_block              = "10.60.1.0/24"
  availability_zone       = "ap-southeast-1a"
  map_public_ip_on_launch = true
  tags                    = { Name = "singapore-peer-subnet" }
}

resource "aws_internet_gateway" "singapore" {
  provider = aws.singapore
  vpc_id   = aws_vpc.singapore.id
  tags     = { Name = "singapore-peer-igw" }
}

resource "aws_route_table" "singapore" {
  provider = aws.singapore
  vpc_id   = aws_vpc.singapore.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.singapore.id
  }
  tags = { Name = "singapore-peer-rt" }
}

resource "aws_route_table_association" "singapore" {
  provider       = aws.singapore
  subnet_id      = aws_subnet.singapore.id
  route_table_id = aws_route_table.singapore.id
}

resource "aws_security_group" "singapore" {
  provider = aws.singapore
  vpc_id   = aws_vpc.singapore.id
  name     = "singapore-peer-sg"

  ingress { from_port = 22 to_port = 22 protocol = "tcp"  cidr_blocks = ["10.50.0.0/16"] }
  ingress { from_port = -1 to_port = -1 protocol = "icmp" cidr_blocks = ["10.50.0.0/16"] }
  egress  { from_port = 0  to_port = 0  protocol = "-1"   cidr_blocks = ["0.0.0.0/0"] }
}

resource "aws_instance" "singapore" {
  provider                = aws.singapore
  ami                      = "ami-0xxxxxxxxxxxxxxxx" # Amazon Linux 2 in ap-southeast-1
  instance_type            = "t2.micro"
  subnet_id                = aws_subnet.singapore.id
  vpc_security_group_ids   = [aws_security_group.singapore.id]
  tags                     = { Name = "singapore-peer-instance" }
}

# --- Peering connection (created from the requester side) ---
resource "aws_vpc_peering_connection" "main" {
  provider      = aws.mumbai
  vpc_id        = aws_vpc.mumbai.id
  peer_vpc_id   = aws_vpc.singapore.id
  peer_region   = "ap-southeast-1"
  tags          = { Name = "mumbai-singapore-pcx" }
}

resource "aws_vpc_peering_connection_accepter" "singapore" {
  provider                  = aws.singapore
  vpc_peering_connection_id = aws_vpc_peering_connection.main.id
  auto_accept                = true
}

resource "aws_route" "mumbai_to_singapore" {
  provider                  = aws.mumbai
  route_table_id             = aws_route_table.mumbai.id
  destination_cidr_block     = "10.60.0.0/16"
  vpc_peering_connection_id  = aws_vpc_peering_connection.main.id
}

resource "aws_route" "singapore_to_mumbai" {
  provider                  = aws.singapore
  route_table_id             = aws_route_table.singapore.id
  destination_cidr_block     = "10.50.0.0/16"
  vpc_peering_connection_id  = aws_vpc_peering_connection.main.id
  depends_on                 = [aws_vpc_peering_connection_accepter.singapore]
}
```

> Note: `auto_accept = true` on the accepter resource only works because both VPCs are in the **same AWS account** — for a true cross-account peering connection, the accepter side would need to run as a separate Terraform apply (or workspace) using the accepter account's credentials, accepting the connection ID passed from the requester's output.

---

## 14. Troubleshooting / Edge Cases Checklist

| Symptom | Likely Cause | Fix |
|---|---|---|
| Can't select the Singapore VPC from a dropdown when creating the peering connection in Mumbai | Console can't browse cross-region resources | Manually paste the Singapore VPC ID (noted in Step B1) |
| Peering connection stuck on `Pending-acceptance` | Nobody has accepted it from the accepter region's console yet | Switch to Singapore console → Peering Connections → Accept Request |
| Peering connection shows `Active` but ping fails | Missing route table entry on one or both sides | Verify both `mumbai-peer-rt` and `singapore-peer-rt` have a route for the *peer's* CIDR pointing at `pcx-xxxx` |
| Route tables correct, still no ping | Missing security group rule on one or both sides | Verify both SGs allow the peer's CIDR inbound for ICMP/SSH |
| Tried to reference the peer's security group ID directly as a source and it's not available | Security-group referencing isn't supported across regions | Use the peer VPC's CIDR block instead — this is expected behavior for inter-region peering, not a bug |
| Peering request disappeared after about a week | Unaccepted requests expire after 168 hours (7 days) | Recreate the peering connection from the requester side |
| Trying to reach a third VPC through the peered VPC | Non-transitive routing — peering doesn't chain | Create a direct peering connection to the third VPC, or use Transit Gateway if you have several VPCs needing mesh connectivity |
| Instance in the peered VPC tried to use the peer's Internet Gateway/NAT/VPN to reach the internet or on-prem | Edge-to-edge routing through a peer's gateway isn't supported | Each VPC must have its own path to the internet/on-prem; peering only carries direct VPC-to-VPC traffic |
| DNS hostname resolves to a public IP instead of private IP when queried from the peer VPC | DNS resolution support not enabled for that direction | Enable it via Peering Connection → Actions → Edit DNS Settings, on the specific side you're querying from |

---

## 15. Interview Q&A

**Q1: What's the difference between VPC Peering and Transit Gateway?**
A: VPC Peering is a direct, point-to-point, non-transitive connection between exactly two VPCs — simple and free to create, but connecting N VPCs together requires N(N-1)/2 individual peering connections (a full mesh), which becomes unmanageable at scale. Transit Gateway is a regional hub-and-spoke router that a large number of VPCs, VPNs, and Direct Connect connections can attach to, with a single control point for routing between all of them — better for many-VPC architectures, at the cost of an hourly attachment charge and slightly more complexity.

**Q2: Why is VPC Peering described as "non-transitive"?**
A: If VPC A peers with B, and A also peers with C, that does **not** mean B can reach C. Each peering connection is a private, isolated relationship between exactly the two VPCs in it — AWS does not route traffic through an intermediate VPC on your behalf, even if that VPC has active peering connections to both endpoints.

**Q3: Can a peered VPC use its peer's Internet Gateway to reach the internet?**
A: No — this is the "edge-to-edge routing" restriction. If VPC A has an Internet Gateway, NAT Gateway, VPN connection, or Direct Connect connection, VPC B cannot use any of those through the peering connection. Each VPC must have its own independent path to the internet or on-premises network.

**Q4: What's required for two VPCs to successfully communicate after a peering connection is Active?**
A: Three things, all done manually — the peering connection must be `Active` (accepted), each VPC's route table must have a static route for the peer's CIDR pointing at the peering connection ID, and each VPC's security groups must explicitly allow the desired traffic from the peer's CIDR (or peer SG ID, if same-region). Missing any one of the three results in a connection that "exists" but doesn't actually pass traffic.

**Q5: Why can't you reference a peer VPC's security group directly as a source in an inter-region peering setup?**
A: AWS's security-group-referencing feature (allowing rules like "allow traffic from sg-xxxx") only works within a single Region's internal networking metadata — it isn't replicated or resolvable across Regions. For inter-region peering, you're limited to CIDR-block-based rules instead.

**Q6: Is VPC Peering traffic encrypted?**
A: For inter-region peering, yes — AWS automatically encrypts traffic crossing Regions using AEAD algorithms on the AWS backbone, with no configuration required. For same-region peering, traffic stays within AWS's own network infrastructure but isn't separately encrypted the way inter-region traffic is, since it never leaves a single Region's physical boundary.

**Q7: How does VPC Peering pricing compare to a Site-to-Site VPN or Client VPN?**
A: VPC Peering itself has no hourly connection charge — you only pay standard data transfer rates for traffic that crosses an AZ or Region, and same-AZ traffic is free. Site-to-Site VPN and Client VPN both have ongoing hourly charges (per connection and per association, respectively) in addition to data transfer, making Peering considerably cheaper for pure VPC-to-VPC connectivity when a full mesh isn't needed.

**Q8: What happens if you don't accept a peering connection request in time?**
A: It expires automatically after 168 hours (7 days) from creation if left in `Pending-acceptance` state, and the requester would need to submit a new request.

---

## 16. Cheat Sheet

```
VPC Peering = direct, non-transitive, private VPC-to-VPC connection (no gateway/VPN/appliance)

Order of creation:
  1. Mumbai: VPC → Subnet → IGW → Route Table → Security Group → EC2
  2. Singapore: VPC → Subnet → IGW → Route Table → Security Group → EC2
  3. Mumbai: Create Peering Connection (paste Singapore's VPC ID manually)
  4. Singapore: Accept the peering connection request
  5. Mumbai route table: add route → Singapore CIDR → target = pcx-xxxx
  6. Singapore route table: add route → Mumbai CIDR → target = pcx-xxxx
  7. Mumbai SG: allow SSH/ICMP inbound from Singapore CIDR
  8. Singapore SG: allow SSH/ICMP inbound from Mumbai CIDR
  9. (Optional) Enable DNS resolution support on both sides
  10. Verify: ping/SSH between private IPs only (not public IPs)

Key facts:
  Non-transitive               → A-B and A-C peering does NOT give B-C connectivity
  No edge-to-edge routing       → can't use peer's IGW/NAT/VPN/Direct Connect
  No route auto-propagation     → must manually add routes on BOTH sides
  SG referencing                → same-region only; inter-region needs CIDR-based rules
  MTU                            → 9001 (jumbo) same-region, 1500 inter-region
  Cost                           → free to create; pay only for cross-AZ/cross-region data transfer
  Expiry                         → unaccepted requests expire after 168 hours (7 days)
  Quota                          → 50 active peering connections per VPC by default (up to 125)
```

---

## 17. Mastery Checklist

- [ ] Explained what VPC Peering is and how it differs from Transit Gateway and Site-to-Site VPN
- [ ] Built both VPCs (Mumbai, Singapore) with non-overlapping CIDRs, subnets, IGWs, route tables, SGs, and EC2 instances
- [ ] Created the peering connection from Mumbai by manually pasting Singapore's VPC ID
- [ ] Accepted the peering connection from the Singapore console
- [ ] Added static routes for the peer's CIDR on BOTH route tables
- [ ] Added SG rules for the peer's CIDR on BOTH security groups (using CIDR, not SG-reference, since this is inter-region)
- [ ] Verified connectivity using **private IPs only** (not public IPs) between the two instances
- [ ] Can explain non-transitive routing and the edge-to-edge routing restriction
- [ ] Can explain why SG-to-SG referencing doesn't work across regions
- [ ] Can explain the MTU difference between same-region and inter-region peering
- [ ] Cleaned up: deleted the peering connection, terminated both instances, removed SGs/route tables/subnets/IGWs/VPCs in both regions

---

## 18. Cleanup (Correct Order Matters)

1. **Delete the VPC Peering Connection** — from either region's console → Peering Connections → select `mumbai-singapore-pcx` → Actions → Delete VPC peering connection.
2. **Terminate both EC2 instances** (Mumbai and Singapore).
3. **Delete both security groups** (`mumbai-peer-sg`, `singapore-peer-sg`).
4. **Delete both route tables** (this also removes the peering routes along with everything else — no need to remove the peering route individually first).
5. **Detach and delete both Internet Gateways.**
6. **Delete both subnets.**
7. **Delete both VPCs** (`mumbai-peer-vpc`, `singapore-peer-vpc`).

> Deleting the peering connection first is not strictly required before deleting the VPCs/route tables (AWS will clean up a peering connection automatically if either VPC is deleted), but deleting it explicitly first keeps the teardown clean and avoids any ambiguity about what depends on what.
