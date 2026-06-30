# AWS Transit Gateway


---


## 1. What is Transit Gateway?

**AWS Transit Gateway (TGW)** is a regional, fully-managed networking service that acts as a **cloud router / hub**. Instead of connecting every VPC to every other VPC individually (mesh peering), each VPC connects **once** to the Transit Gateway, and the Transit Gateway routes traffic between all attached networks.

Think of it like an **airport hub**: instead of every city having a direct flight to every other city, all flights go through one central hub, and the hub routes passengers onward. TGW does the same thing for network packets between VPCs, VPNs, Direct Connect, and even other AWS accounts.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/35efea70-a3e4-4bb4-81d8-6a4f48cea3af" />

---

## 2. Why Use Transit Gateway?

| Problem (without TGW) | How TGW Solves It |
|---|---|
| VPC peering doesn't scale — connections grow as N(N-1)/2 | TGW scales linearly — 1 attachment per VPC |
| No transitive routing in VPC peering (A↔B, B↔C does NOT mean A↔C) | TGW provides **transitive routing** by default |
| Managing dozens of peering connections and route tables is error-prone | Centralized route tables, single control point |
| Connecting on-prem (VPN/Direct Connect) to many VPCs needs many connections | One Direct Connect/VPN attached to TGW reaches all VPCs |
| Multi-account, multi-region connectivity is hard to standardize | TGW supports cross-account sharing (RAM) and peering across regions |

**Common use cases:**
- Hub-and-spoke architecture for multiple VPCs (dev, staging, prod, shared-services)
- Centralized egress (single NAT Gateway / firewall VPC for internet-bound traffic)
- Connecting on-premises data centers to multiple VPCs via one VPN/Direct Connect
- Multi-account landing zones (AWS Organizations + RAM sharing)
- Network segmentation (isolating prod from dev using separate route tables)

---

## 3. How It Works — Core Concepts

### 3.1 Transit Gateway (TGW)
The central, regional router resource. You create one TGW per region (per environment, typically).

### 3.2 Transit Gateway Attachment
A connection point between TGW and a network resource. Types of attachments:
- **VPC attachment** — connects a VPC (via subnets, one per AZ) to the TGW
- **VPN attachment** — connects an on-prem Site-to-Site VPN
- **Direct Connect Gateway attachment** — connects a DX private connection
- **Peering attachment** — connects two TGWs (often cross-region)

### 3.3 Transit Gateway Route Table
Each TGW has its own route table(s), **separate from VPC route tables**. This is the most important concept for beginners to internalize:

```
VPC Route Table          TGW Route Table          VPC Route Table
(in VPC-A)                (in Transit GW)           (in VPC-B)

10.1.0.0/16  -> local     10.2.0.0/16 -> Attach-B    10.2.0.0/16 -> local
0.0.0.0/0    -> TGW       10.1.0.0/16 -> Attach-A    0.0.0.0/0   -> TGW
                          10.3.0.0/16 -> Attach-C
                          10.4.0.0/16 -> Attach-D
```

There are **two layers of routing** for any cross-VPC packet:
1. **VPC route table** says "send non-local traffic to the TGW"
2. **TGW route table** says "for this destination CIDR, send out via this attachment"

### 3.4 Association vs Propagation
- **Association**: which TGW route table an attachment uses to route ITS outbound traffic
- **Propagation**: whether an attachment's CIDR is automatically advertised INTO a TGW route table

You can have **multiple TGW route tables** for segmentation — e.g., a "prod" route table that only sees prod VPCs, and a "dev" route table that only sees dev VPCs, even though all attach to the same physical TGW.

### 3.5 Architecture Diagram (4-VPC Hub & Spoke)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/d8819727-3aa0-4dc3-85ae-cc420c6be753" />


---

## 4. Advantages

- **Simplified connectivity** — one attachment per VPC instead of a peering mesh
- **Transitive routing** — VPC-A can reach VPC-C through the TGW without a direct connection
- **Centralized control** — route tables, attachments, and policies managed in one place
- **Scales to thousands of VPCs/attachments** (subject to quotas, which are raisable)
- **Multicast support** (for specific workloads like media streaming)
- **Cross-account and cross-region** connectivity via Resource Access Manager (RAM) and TGW peering
- **Network segmentation** using multiple TGW route tables (e.g., isolate prod from dev on the same TGW)
- **Integrates** with VPN, Direct Connect, and SD-WAN appliances

---

## 5. Disadvantages / Limitations

- **Cost** — hourly charge per attachment + per-GB data processing charge (unlike free VPC peering)
- **Added latency** — traffic takes an extra hop through the TGW compared to direct peering
- **Regional resource** — by default scoped to one region; cross-region needs TGW peering (extra complexity/cost)
- **Bandwidth per attachment** — each VPC attachment has a throughput ceiling (burstable, but a soft limit exists per attachment)
- **Quota limits** — default limits on attachments, route tables, and routes per table (raisable via support ticket)
- **No transitive peering for TGW peering attachments** — TGW-to-TGW peering connections are NOT transitive (a third TGW won't automatically get routes through a peered TGW without explicit route propagation/static routes)
- **Complexity at scale** — large numbers of route tables/associations require strong IaC and naming discipline to avoid misconfiguration

---

## 6. Pricing Overview

> ⚠️ Always confirm current figures on the official AWS Transit Gateway pricing page — these are placeholders to understand the *model*, not real numbers.

| Cost Component | Description |
|---|---|
| **Attachment hourly charge** | Charged per attachment (VPC, VPN, DX, peering) per hour — approx. `$[TGW_ATTACHMENT_HOURLY_RATE]`/hour per attachment |
| **Data processing charge** | Charged per GB of data processed through the TGW — approx. `$[TGW_DATA_PROCESSING_RATE]`/GB |
| **Cross-region peering** | Data transferred across a TGW peering attachment is billed at **inter-region data transfer rates**, in addition to TGW processing charges |
| **No charge for** | Creating the TGW itself (no resource is billed until attachments exist) |

**Cost optimization tips:**
- Consolidate east-west traffic patterns to avoid unnecessary cross-AZ/cross-region data processing
- Use **centralized egress** (single NAT Gateway via TGW) to reduce duplicate NAT Gateway costs across VPCs
- Monitor data processed via CloudWatch metric `BytesIn`/`BytesOut` per attachment to spot cost anomalies

---

## 7. Assumptions

For this guide's lab and architecture sections, the following are assumed:

- You have an AWS account with sufficient IAM permissions (VPC, EC2, TGW full access)
- Working region: `[AWS_REGION]` (e.g., `ap-south-1`)
- Account ID placeholder: `[ACCOUNT_ID]`
- All 4 VPCs are created in the **same region** (no cross-region peering in the base lab — covered separately in production section)
- EC2 instances use **Amazon Linux 2023**, free-tier eligible type (e.g., `t3.micro` / `t2.micro`) — replace with `[EC2_INSTANCE_TYPE]` as needed
- Security Groups will be used to control instance-level access; TGW route tables control network-level reachability
- You will use an existing or newly created **EC2 key pair**: `[KEY_PAIR_NAME]`
- No NAT Gateway / Internet Gateway is required for the core lab (private connectivity test between VPCs only) — internet access is optional and noted separately
- CIDR blocks across the 4 VPCs **must not overlap** (this is a hard TGW requirement)

---

## 8. Practical Lab Guide — 4 VPCs, 4 EC2 Each

### 8.1 Lab Objective
Build a hub-and-spoke network where 4 VPCs (each with 4 EC2 instances) all communicate with each other through a single Transit Gateway — built entirely via the **AWS Management Console**, validated with `ping`/`curl` between instances across VPCs.

### 8.2 CIDR Plan (non-overlapping — required)

| Resource | CIDR Placeholder | Example |
|---|---|---|
| VPC-A | `[VPC_CIDR_1]` | `10.1.0.0/16` |
| VPC-B | `[VPC_CIDR_2]` | `10.2.0.0/16` |
| VPC-C | `[VPC_CIDR_3]` | `10.3.0.0/16` |
| VPC-D | `[VPC_CIDR_4]` | `10.4.0.0/16` |
| Each VPC's EC2 subnet | `/24` from its VPC CIDR | e.g. `10.1.1.0/24` |

> 🔑 **Golden Rule**: TGW does NOT do NAT or CIDR translation. If two VPCs have overlapping CIDRs, TGW **cannot** route between them. Always plan non-overlapping address space *before* creating VPCs.

> 🖱️ This lab is written entirely for the **AWS Management Console** — no CLI required. Open the console in your browser and confirm you're in the correct region (`[AWS_REGION]`) using the region dropdown in the top-right corner before starting.

### 8.3 Step 1 — Create the 4 VPCs (Console)

Navigate to **VPC Console → Your VPCs → Create VPC**.

Repeat this **4 times** (once per VPC), using **"VPC only"** (not the "VPC and more" wizard, so you control subnet creation yourself in Step 1b):

1. **Name tag**: `VPC-A` (then `VPC-B`, `VPC-C`, `VPC-D` for the others)
2. **IPv4 CIDR block**: `[VPC_CIDR_1]` → e.g. `10.1.0.0/16` (use `[VPC_CIDR_2]`, `[VPC_CIDR_3]`, `[VPC_CIDR_4]` for B/C/D — e.g. `10.2.0.0/16`, `10.3.0.0/16`, `10.4.0.0/16`)
3. **IPv6 CIDR block**: No IPv6 CIDR block (leave default)
4. **Tenancy**: Default
5. Click **Create VPC**

> 📝 After each VPC is created, note its **VPC ID** (e.g., `vpc-0abc123...`) — you'll select it by name in later dropdowns, so consistent naming matters more than memorizing IDs.

**Step 1b — Create a Subnet in each VPC**

Go to **VPC Console → Subnets → Create subnet**:

1. **VPC ID**: select `VPC-A`
2. **Subnet name**: `VPC-A-Subnet`
3. **Availability Zone**: pick any AZ in `[AWS_REGION]` (e.g., `[AWS_REGION]a`)
4. **IPv4 CIDR block**: a `/24` from VPC-A's range → e.g. `10.1.1.0/24`
5. Click **Create subnet**

Repeat for VPC-B (`10.2.1.0/24`), VPC-C (`10.3.1.0/24`), and VPC-D (`10.4.1.0/24`).

> 🔑 **Golden Rule**: TGW does NOT do NAT or CIDR translation. If two VPCs have overlapping CIDRs, TGW **cannot** route between them. Double-check your 4 CIDR blocks don't overlap before moving on.

### 8.4 Step 2 — Create the Transit Gateway (Console)

Navigate to **VPC Console → Transit Gateways → Transit Gateways → Create Transit Gateway**.

1. **Name tag**: `Lab-TGW`
2. **Description**: `Lab-TGW`
3. **Amazon side ASN**: leave default (`64512`)
4. **DNS support**: Enable (recommended)
5. **VPN ECMP support**: leave default
6. **Default route table association**: ✅ Enable
7. **Default route table propagation**: ✅ Enable
8. **Auto accept shared attachments**: Disable (leave default)
9. Click **Create Transit Gateway**

Wait 1–2 minutes for the **State** column to change from `pending` to `available` on the Transit Gateways list page.

> 💡 Leaving **Default route table association/propagation** enabled means every attachment automatically uses the single built-in TGW route table — the simplest setup for beginners. (Production uses these **disabled** + custom route tables — covered in Section 9.)

### 8.5 Step 3 — Attach Each VPC to the TGW (Console)

Navigate to **VPC Console → Transit Gateways → Transit Gateway Attachments → Create Transit Gateway Attachment**.

Repeat **4 times** (once per VPC):

1. **Transit Gateway ID**: select `Lab-TGW`
2. **Attachment type**: `VPC`
3. **Attachment name tag**: `Attach-VPC-A` (then B/C/D)
4. **DNS support**: Enable
5. **VPC ID**: select `VPC-A`
6. **Subnets**: check the box next to `VPC-A-Subnet` (select one subnet per AZ you created)
7. Click **Create Transit Gateway Attachment**

Repeat for VPC-B, VPC-C, VPC-D. Wait for each attachment's **State** to show `Available` on the Transit Gateway Attachments list (refresh the page periodically).

### 8.6 Step 4 — Update Each VPC's Route Table (Console)

Navigate to **VPC Console → Route Tables**, find the route table associated with **VPC-A's** subnet (usually the "main" route table unless you created a custom one).

1. Select the route table → **Routes** tab → **Edit routes** → **Add route**
2. **Destination**: `[VPC_CIDR_2]` (e.g. `10.2.0.0/16`) → **Target**: select `Transit Gateway` → choose `Lab-TGW`
3. **Add route** again → **Destination**: `[VPC_CIDR_3]` → **Target**: `Lab-TGW`
4. **Add route** again → **Destination**: `[VPC_CIDR_4]` → **Target**: `Lab-TGW`
5. Click **Save changes**

Repeat this for **VPC-B's, VPC-C's, and VPC-D's** route tables — each pointing to the **other three** CIDRs via `Lab-TGW`.

> ✅ Because **Default route table propagation** was enabled in Step 2, the TGW's internal route table already knows about all 4 VPC CIDRs automatically — you only need to edit the **VPC-level** route tables above, not the TGW route table itself, for this basic lab.

### 8.7 Step 5 — Security Groups for Cross-VPC Testing (Console)

Navigate to **EC2 Console → Security Groups → Create security group** (or edit the default SG) — repeat per VPC:

1. **Security group name**: `VPC-A-SG`
2. **VPC**: select `VPC-A`
3. **Inbound rules** → **Add rule**:
   - Type: `All ICMP - IPv4`, Source: Custom → `[VPC_CIDR_2]`
   - Add rule → Type: `All ICMP - IPv4`, Source: Custom → `[VPC_CIDR_3]`
   - Add rule → Type: `All ICMP - IPv4`, Source: Custom → `[VPC_CIDR_4]`
   - Add rule → Type: `SSH`, Port `22`, Source: Custom → `[VPC_CIDR_2]` (repeat for CIDR_3 and CIDR_4 if you want SSH testing too)
4. Click **Create security group**

Repeat for VPC-B-SG, VPC-C-SG, VPC-D-SG, each allowing ICMP/SSH from the **other three** VPC CIDRs.

### 8.8 Step 6 — Launch 4 EC2 Instances per VPC (16 Total) (Console)

Navigate to **EC2 Console → Instances → Launch instances**. Repeat this whole flow 4 times (once per VPC):

1. **Name**: `VPC-A-Instance` (the console will auto-number them 1–4 if you set "Number of instances" below)
2. **AMI**: Amazon Linux 2023 (free-tier eligible)
3. **Instance type**: `[EC2_INSTANCE_TYPE]` (e.g. `t3.micro` / `t2.micro`, free-tier eligible)
4. **Key pair**: select `[KEY_PAIR_NAME]` (or create a new one)
5. **Network settings** → **Edit**:
   - **VPC**: `VPC-A`
   - **Subnet**: `VPC-A-Subnet`
   - **Auto-assign public IP**: Disable (this lab uses private connectivity only)
   - **Firewall (security group)**: select existing → `VPC-A-SG`
6. **Number of instances**: `4`
7. Click **Launch instance**

Repeat for VPC-B, VPC-C, VPC-D (using their respective subnet/SG) — giving **16 EC2 instances total (4 × 4)**.

> 💡 Since instances have **no public IP** and there's no Internet Gateway/NAT in this lab, use **EC2 Instance Connect Endpoint** or **SSM Session Manager** (via VPC interface endpoints) to access instances from the console, rather than SSH over the public internet. To connect via Instance Connect: select the instance → **Connect** button → **EC2 Instance Connect** tab (you may need to create an Instance Connect Endpoint in that VPC first under **VPC Console → Endpoints**).

### 8.9 Step 7 — Validate Connectivity (Console)

For each instance, note its **Private IPv4 address** from the **EC2 Console → Instances** list (click the instance → details panel).

1. Connect to an instance in **VPC-A** (via EC2 Instance Connect or Session Manager — both open a browser-based terminal, no SSH client needed)
2. From the terminal that opens, ping/curl instances in the other VPCs using their **private IPs**:

```bash
ping -c 4 [VPC_B_INSTANCE_PRIVATE_IP]
ping -c 4 [VPC_C_INSTANCE_PRIVATE_IP]
ping -c 4 [VPC_D_INSTANCE_PRIVATE_IP]
```

Expected result: successful replies, confirming traffic flows VPC-A → TGW → VPC-B/C/D.

> Note: this is just running `ping` on the instance's own OS shell (same as on a laptop) — it isn't "using the AWS CLI" to manage AWS resources.

### 8.10 Step 8 — Inspect the TGW Route Table (Verification, Console)

Navigate to **VPC Console → Transit Gateways → Transit Gateway Route Tables**.

1. Select the route table associated with `Lab-TGW` (the default one)
2. Click the **Routes** tab
3. You should see all 4 VPC CIDRs (`[VPC_CIDR_1]` through `[VPC_CIDR_4]`) listed, each with **Route type** = `Propagated` and a **Resource ID** pointing to the matching attachment

This confirms the TGW has automatically learned all 4 VPC CIDRs via propagation.

### 8.11 Lab Summary Diagram

<img width="1484" height="1060" alt="image" src="https://github.com/user-attachments/assets/17687ff3-69a7-4fa5-9adf-2166241d46cc" />


---

## 9. Production-Level Architecture

### 9.1 Key Differences from the Lab

| Lab (Basic) | Production |
|---|---|
| Single default TGW route table | **Multiple custom route tables** for segmentation (prod, dev, shared-services) |
| Auto-propagation for everything | **Explicit association + propagation** per attachment — least-privilege routing |
| Single subnet/AZ per VPC | **Multiple subnets across 2-3 AZs** per VPC for resilience |
| No centralized egress | **Centralized egress VPC** with NAT Gateway, shared via TGW |
| No inspection | **Centralized inspection VPC** with firewall appliances (e.g., AWS Network Firewall) in the traffic path |
| Single account | **Multi-account** via AWS Organizations + Resource Access Manager (RAM) sharing |
| No monitoring | **CloudWatch + VPC Flow Logs + TGW Flow Logs** for observability |

### 9.2 Segmented Route Table Design

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/cc8af1b1-a370-467a-8a5e-255739303a11" />


### 9.3 Centralized Egress Pattern

<img width="1323" height="1189" alt="image" src="https://github.com/user-attachments/assets/6cc673dc-b8a3-49a8-bcb4-c1b7fd7aaad9" />


This avoids deploying a NAT Gateway in every spoke VPC (cost savings) and gives a **single choke point** for security inspection/logging of all outbound traffic.

### 9.4 Multi-Region Design (TGW Peering)

<img width="1774" height="887" alt="image" src="https://github.com/user-attachments/assets/d38e6b71-3b5d-400a-9604-e9c48a88c2a5" />


- TGW peering attachments are **not transitive** — explicit static routes are required in each TGW's route table pointing to the other region's CIDRs via the peering attachment.
- Inter-region data transfer charges apply on top of standard TGW data processing charges.

### 9.5 Multi-Account Sharing (RAM)
<img width="1692" height="929" alt="image" src="https://github.com/user-attachments/assets/1fc33570-a31b-46f8-bebd-8f76be7cbe9c" />


### 9.6 Resilience Considerations

- Attach VPCs via subnets in **at least 2 AZs** so the TGW attachment survives an AZ failure
- Use **TGW Flow Logs** + CloudWatch alarms on attachment-level `BytesDropped` metrics to catch routing/black-hole issues early
- Apply **Infrastructure as Code (Terraform)** for all TGW resources to prevent drift and enable peer review of route changes

### 9.7 Terraform Skeleton (Production Pattern)

```hcl
resource "aws_ec2_transit_gateway" "main" {
  description                    = "Production-TGW"
  amazon_side_asn                 = 64512
  default_route_table_association = "disable"
  default_route_table_propagation = "disable"
  tags = { Name = "Production-TGW" }
}

resource "aws_ec2_transit_gateway_route_table" "prod" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  tags = { Name = "Prod-RT" }
}

resource "aws_ec2_transit_gateway_route_table" "dev" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  tags = { Name = "Dev-RT" }
}

resource "aws_ec2_transit_gateway_vpc_attachment" "prod_vpc1" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  vpc_id             = var.prod_vpc1_id
  subnet_ids         = var.prod_vpc1_subnet_ids
  tags = { Name = "Attach-Prod-VPC1" }
}

resource "aws_ec2_transit_gateway_route_table_association" "prod_vpc1" {
  transit_gateway_attachment_id = aws_ec2_transit_gateway_vpc_attachment.prod_vpc1.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.prod.id
}

resource "aws_ec2_transit_gateway_route_table_propagation" "prod_vpc1" {
  transit_gateway_attachment_id = aws_ec2_transit_gateway_vpc_attachment.prod_vpc1.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.prod.id
}
```

> Replicate the attachment/association/propagation block per VPC, and add explicit `aws_ec2_transit_gateway_route` resources where static routing (rather than propagation) is required — e.g., routing to the shared egress VPC from both prod and dev route tables.

---

## 10. Edge Cases

| Edge Case | What Happens | How to Handle |
|---|---|---|
| **Overlapping CIDRs** between two VPCs | TGW cannot route between them — routing fails silently or routes to the wrong VPC | Always plan a non-overlapping IP address scheme org-wide before attaching VPCs |
| **Black-holed routes** (attachment deleted but route remains) | Traffic to that CIDR is silently dropped | Monitor `BlackholeRouteCount`/Flow Logs; clean up routes when deleting attachments |
| **Asymmetric routing** (VPC-A → VPC-B works, but VPC-B → VPC-A doesn't) | One-way connectivity issues, often seen with stateful firewalls | Verify route tables AND security groups/NACLs on **both** sides |
| **TGW peering is not transitive** | VPC in Region-1 cannot reach VPC in Region-3 just because Region-1↔Region-2 and Region-2↔Region-3 are peered | Add explicit static routes for every required path, or use a full-mesh of TGW peering |
| **Attachment in `pending acceptance` state** (cross-account) | Traffic won't flow until the attachment is accepted by the TGW owner account | Use `accept-transit-gateway-vpc-attachment` or enable `AutoAcceptSharedAttachments` cautiously |
| **Exceeding route table quotas** | New routes fail to propagate once the per-route-table limit is hit | Monitor quotas via Service Quotas console; request increases proactively |
| **Bandwidth bottleneck per attachment** | High-throughput workloads see throttling despite TGW being "highly available" | Distribute load across multiple attachments/ENIs, or re-architect with Direct Connect for sustained high-throughput links |
| **DNS resolution across VPCs** | Private hosted zones are NOT automatically shared via TGW | Use Route 53 Resolver rules/endpoints, or VPC association authorization, in parallel with TGW |
| **Security Group reference across VPCs** | SGs cannot reference SG IDs from a different VPC, even when connected via TGW | Use CIDR-based rules instead of SG-ID references for cross-VPC traffic |

---

## 11. Best Practices

- Use **separate TGW route tables** per trust boundary (prod/dev/shared) rather than relying on the single default route table in production
- Always disable `DefaultRouteTableAssociation`/`Propagation` in production and manage associations/propagations explicitly
- Centralize egress and inspection in a dedicated **shared-services VPC**
- Tag every TGW resource (attachment, route table) consistently for cost allocation and auditing
- Enable **TGW Flow Logs** to S3/CloudWatch for traffic visibility and troubleshooting
- Manage all TGW infrastructure via **Terraform/CloudFormation** — never click-ops in production
- Document your CIDR allocation plan centrally (e.g., IPAM) before adding new VPCs to avoid future overlap

---

## 12. Interview Q&A + Cheat Sheet

**Q1: What problem does Transit Gateway solve that VPC Peering doesn't?**
A: VPC Peering is non-transitive and requires a full mesh of connections (N² growth). TGW provides transitive routing through a central hub with linear (N) growth in connections.

**Q2: Is traffic between TGW route tables transitive by default?**
A: Routing is determined by association + propagation, not automatically transitive — you must explicitly design which route tables see which attachments.

**Q3: Is TGW peering transitive?**
A: No. TGW-to-TGW peering attachments are not transitive; explicit static routes are required for multi-hop paths across peered TGWs.

**Q4: How do you isolate prod and dev traffic on the same Transit Gateway?**
A: Create separate TGW route tables for prod and dev, associate each VPC's attachment with only its own route table, and only propagate routes within the same trust boundary (with shared-services routes added explicitly where needed).

**Q5: What causes traffic to silently drop after deleting a VPC attachment?**
A: A black-holed route — the TGW route table still references the deleted attachment as a target. Must be manually removed.

**Q6: Can overlapping CIDRs be connected via TGW?**
A: No — TGW does not perform NAT/CIDR translation. CIDRs across attached VPCs must be unique.

**Cheat Sheet:**

```
TGW          = Regional hub router
Attachment   = Connection point (VPC / VPN / DX / Peering)
TGW RT       = Routing table living inside TGW (separate from VPC RT)
Association  = Which TGW RT an attachment uses for its outbound routing
Propagation  = Whether an attachment's CIDR is advertised into a TGW RT
Transitive   = TGW routes between VPCs natively; TGW peering does NOT
Black hole   = Route pointing to a deleted/unavailable attachment
```

---

## 13. Cleanup

To avoid ongoing charges after the lab, tear down resources in this order using the **Console**:

1. **Terminate all EC2 instances**: **EC2 Console → Instances** → select all 16 lab instances → **Instance state → Terminate instance**
2. **Delete TGW VPC attachments**: **VPC Console → Transit Gateways → Transit Gateway Attachments** → select each of the 4 attachments → **Actions → Delete Transit Gateway Attachment** → wait for state `Deleted`
3. **Delete the Transit Gateway**: **VPC Console → Transit Gateways → Transit Gateways** → select `Lab-TGW` → **Actions → Delete Transit Gateway**
4. **Delete subnets**: **VPC Console → Subnets** → select each of the 4 lab subnets → **Actions → Delete subnet**
5. **Delete VPCs**: **VPC Console → Your VPCs** → select each of the 4 VPCs → **Actions → Delete VPC**

> ⚠️ Deleting a TGW fails if attachments still exist — always delete attachments first and wait for the `Deleted` state before deleting the TGW itself. Likewise, a VPC can't be deleted while its attachment or instances still reference it.

---

