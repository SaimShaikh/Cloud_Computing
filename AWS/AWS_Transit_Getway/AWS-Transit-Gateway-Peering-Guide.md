# AWS Transit Gateway Peering — Complete Beginner-to-Production Guide

> A hands-on, **AWS Console-based** guide covering Transit Gateway (TGW) Peering — connecting two Transit Gateways across regions (or accounts) so VPCs attached to each can talk to each other.

---

## Table of Contents

1. [What is Transit Gateway Peering?](#1-what-is-transit-gateway-peering)
2. [Why Use Transit Gateway Peering?](#2-why-use-transit-gateway-peering)
3. [How It Works — Core Concepts](#3-how-it-works--core-concepts)
4. [Advantages](#4-advantages)
5. [Disadvantages / Limitations](#5-disadvantages--limitations)
6. [Pricing Overview](#6-pricing-overview)
7. [Assumptions](#7-assumptions)
8. [Practical Lab Guide — Peering 2 TGWs Across Regions (Console)](#8-practical-lab-guide--peering-2-tgws-across-regions-console)
9. [Production-Level Architecture](#9-production-level-architecture)
10. [Edge Cases](#10-edge-cases)
11. [Best Practices](#11-best-practices)
12. [Interview Q&A + Cheat Sheet](#12-interview-qa--cheat-sheet)
13. [Cleanup](#13-cleanup)

---

## 1. What is Transit Gateway Peering?

**Transit Gateway Peering** is a connection between **two Transit Gateways** that lets resources (VPCs, VPNs, Direct Connect) attached to one TGW communicate with resources attached to the other TGW — most commonly **across two different AWS Regions**, but also usable across two different **AWS accounts** in the same region.

Think of it like connecting two separate "hub airports" with a direct long-haul flight: each hub still manages its own local routes (its own attached VPCs), but the **peering connection** is the single link that lets traffic cross from one hub's network to the other's.

```
   Region: ap-south-1                         Region: us-east-1
   ┌─────────────────────────┐                ┌─────────────────────────┐
   │   Transit Gateway-1       │   Peering     │   Transit Gateway-2       │
   │   (Mumbai)                │◄─────────────►│   (N. Virginia)           │
   │                           │  Attachment    │                           │
   │   Attach-VPC-A            │                │   Attach-VPC-C            │
   │   Attach-VPC-B            │                │   Attach-VPC-D            │
   └─────────────────────────┘                └─────────────────────────┘
            │       │                                   │       │
         VPC-A    VPC-B                              VPC-C    VPC-D
```

A TGW peering connection is itself a special kind of **attachment** — each TGW sees the peering as an attachment in its own route table, exactly like a VPC attachment.

---

## 2. Why Use Transit Gateway Peering?

| Problem | How TGW Peering Solves It |
|---|---|
| Need connectivity between VPCs in **different regions** (e.g., disaster recovery region, multi-region active-active app) | TGW Peering links two regional TGWs with a single connection instead of cross-region VPC peering for every VPC pair |
| Multiple AWS accounts/business units each run their own regional TGW, but need limited cross-environment connectivity | Peer the two TGWs and selectively route only the required CIDRs |
| Want to avoid the N² mesh of cross-region VPC peering connections | One peering attachment per region-pair covers all VPCs attached to each TGW (once routes are added) |
| Need encrypted, private backbone connectivity between regions without managing a VPN | TGW peering traffic travels over the **AWS global backbone**, not the public internet |

**Common use cases:**
- Multi-region disaster recovery (DR) — primary region TGW peered to standby region TGW
- Global applications with regional VPCs that need limited, controlled cross-region access
- Mergers/acquisitions — peering two separate organizations' TGWs (different accounts) for shared services
- Centralized security/shared-services VPC in one region reachable from spoke VPCs in another region

---

## 3. How It Works — Core Concepts

### 3.1 Peering Attachment
A TGW peering attachment is a **point-to-point** connection between exactly two TGWs. One side **requests** the peering, the other side **accepts** it (similar to VPC peering's request/accept model).

### 3.2 Requester and Accepter
- **Requester TGW**: initiates the peering connection (in Region/Account A)
- **Accepter TGW**: must explicitly accept the peering request before traffic can flow (in Region/Account B)

### 3.3 Non-Transitive by Design
This is the **most important concept** to internalize:

> 🔑 **TGW peering attachments are NOT transitive.** If TGW-1 is peered with TGW-2, and TGW-2 is peered with TGW-3, **TGW-1 cannot automatically reach TGW-3** through TGW-2. You must either add a direct TGW-1↔TGW-3 peering connection, or explicitly engineer routing (which AWS does not support as automatic transit routing through a middle TGW for peering).

```
   TGW-1 ◄──peered──► TGW-2 ◄──peered──► TGW-3

   TGW-1 CAN reach TGW-2's VPCs   ✅
   TGW-2 CAN reach TGW-3's VPCs   ✅
   TGW-1 CANNOT reach TGW-3's VPCs automatically  ❌
   (no direct peering + no transit-through-TGW-2)
```

### 3.4 Static Routes Only (No Propagation Across Peering, by default behavior to know)
Within a single TGW, VPC attachments can use **automatic route propagation**. For peering attachments, you typically add **static routes** in each TGW's route table pointing specific remote CIDRs to the peering attachment as the target. (Propagation CAN be enabled across a peering attachment too, but many learners start with static routes since it makes the explicit, intentional nature of cross-region routing clearer.)

```
TGW-1 Route Table (Mumbai)             TGW-2 Route Table (N. Virginia)

[VPC_CIDR_1] -> Attach-VPC-A           [VPC_CIDR_3] -> Attach-VPC-C
[VPC_CIDR_2] -> Attach-VPC-B           [VPC_CIDR_4] -> Attach-VPC-D
[VPC_CIDR_3] -> Peering-Attachment     [VPC_CIDR_1] -> Peering-Attachment
[VPC_CIDR_4] -> Peering-Attachment     [VPC_CIDR_2] -> Peering-Attachment
   (static route, added manually)         (static route, added manually)
```

### 3.5 ASN Requirement
Each TGW has an **Amazon-side ASN** (Autonomous System Number). The two TGWs being peered must have **different ASNs** if you customized them — using the AWS default (`64512`) on both sides usually still works for peering since BGP/ASN conflict only matters in specific dynamic-routing contexts, but best practice is to assign **unique ASNs per region/TGW** to avoid any future conflicts (e.g., `64512` for Region-1, `64513` for Region-2).

### 3.6 Full Architecture Diagram

```
        Region A (e.g. ap-south-1)                    Region B (e.g. us-east-1)
   ┌───────────────────────────────┐             ┌───────────────────────────────┐
   │                                 │             │                                 │
   │   ┌─────────────────────┐       │   Peering   │       ┌─────────────────────┐   │
   │   │  Transit Gateway-1    │◄─────┼─────────────┼──────►│  Transit Gateway-2    │   │
   │   │  ASN: 64512           │      │ Attachment  │       │  ASN: 64513           │   │
   │   └──────────┬──────────┘       │             │       └──────────┬──────────┘   │
   │      Attach-A │   Attach-B       │             │      Attach-C │   Attach-D      │
   │         ┌─────┴─────┐ ┌──────┐   │             │   ┌──────┐ ┌─────┴─────┐         │
   │         │   VPC-A    │ │VPC-B │   │             │   │VPC-C │ │   VPC-D    │         │
   │         │[VPC_CIDR_1]│ │[..2] │   │             │   │[..3] │ │[VPC_CIDR_4]│         │
   │         └───────────┘ └──────┘   │             │   └──────┘ └───────────┘         │
   └───────────────────────────────┘             └───────────────────────────────┘
```

---

## 4. Advantages

- **No public internet exposure** — peering traffic travels over AWS's private global backbone
- **No N² VPC peering mesh** across regions — one TGW peering connection can carry traffic for all attached VPCs (once routed)
- **Works across accounts** — supports peering between TGWs owned by different AWS accounts (useful for M&A, multi-org setups)
- **Encryption in transit** — traffic across TGW peering attachments is automatically encrypted (no extra configuration needed)
- **Simple conceptual model** — same attachment/route-table pattern as VPC attachments, just peering-type instead

---

## 5. Disadvantages / Limitations

- **Not transitive** — peering does not chain; a "TGW-1 → TGW-2 → TGW-3" path does not work without a direct TGW-1↔TGW-3 peering connection
- **Manual/static routing burden** — since automatic propagation across peering is often skipped by beginners (or limited in some designs), routes must be added and maintained manually on both sides
- **Inter-region data transfer cost** — traffic over a peering connection between regions is billed at AWS inter-region data transfer rates, in addition to standard TGW data processing charges
- **One peering connection per TGW pair** — to fully mesh 3+ regions, you need a peering connection for **every pair** of TGWs (N(N-1)/2 growth again, but at the TGW level, not the VPC level — much smaller scale than VPC-level mesh)
- **No support for multicast** over peering attachments
- **Accepter must actively accept** — peering stays in `pending-acceptance` state until manually approved, which can stall automation if not handled in IaC

---

## 6. Pricing Overview

> ⚠️ Always confirm current figures on the official AWS Transit Gateway pricing page — these are placeholders to understand the *model*, not real numbers.

| Cost Component | Description |
|---|---|
| **Peering attachment hourly charge** | Each side's peering attachment is billed per hour, similar to a VPC attachment — approx. `$[TGW_PEERING_ATTACHMENT_HOURLY_RATE]`/hour |
| **Data processing charge** | Per-GB charge for data processed through the TGW on each side — approx. `$[TGW_DATA_PROCESSING_RATE]`/GB |
| **Inter-region data transfer** | Data crossing the peering connection between two regions is billed at standard **inter-region data transfer** rates (varies by region pair) — approx. `$[INTER_REGION_TRANSFER_RATE]`/GB |
| **No charge for** | The peering *request* itself — billing starts once the attachment is active and processing traffic |

**Cost optimization tips:**
- Only route the specific CIDRs that genuinely need cross-region access — avoid blanket `0.0.0.0/0` style routes across peering connections to limit unnecessary data processing charges
- Monitor cross-region data volume via CloudWatch metrics per TGW attachment to catch runaway costs (e.g., a misconfigured backup job replicating across regions unintentionally)
- Consider whether the workload truly needs cross-region TGW peering vs. a regional read-replica/CDN pattern that avoids recurring cross-region traffic altogether

---

## 7. Assumptions

For this guide's lab and architecture sections, the following are assumed:

- You already have (or will create) **two separate Transit Gateways**, each in a different AWS Region: `[AWS_REGION_1]` (e.g. `ap-south-1`) and `[AWS_REGION_2]` (e.g. `us-east-1`)
- Each TGW already has at least **one VPC attached** (you can reuse VPC-A/VPC-B from the base Transit Gateway lab as the Region-1 side, and create 1–2 new VPCs in Region-2 for this lab)
- Account ID placeholder: `[ACCOUNT_ID]` (same account for both TGWs in the base lab; cross-account variant is covered in Section 9)
- CIDR blocks across **all** VPCs in both regions are non-overlapping (hard requirement, same as standard TGW)
- You are using the **AWS Management Console** exclusively — no AWS CLI required
- EC2 instances (if testing end-to-end connectivity) use Amazon Linux 2023, free-tier eligible type `[EC2_INSTANCE_TYPE]`, accessed via EC2 Instance Connect or SSM Session Manager (no public IP / no SSH over internet)
- TGW ASNs are set to distinct values: `[TGW1_ASN]` (e.g. `64512`) and `[TGW2_ASN]` (e.g. `64513`)

---

## 8. Practical Lab Guide — Peering 2 TGWs Across Regions (Console)

### 8.1 Lab Objective
Create two Transit Gateways in two different regions, peer them together, add static routes on both sides, and validate that an EC2 instance in a Region-1 VPC can reach an EC2 instance in a Region-2 VPC through the peering connection.

### 8.2 Topology for This Lab

```
   Region-1: [AWS_REGION_1]                      Region-2: [AWS_REGION_2]
   ┌──────────────────────┐                      ┌──────────────────────┐
   │  TGW-1 (ASN [TGW1_ASN]) │◄────Peering────────►│  TGW-2 (ASN [TGW2_ASN]) │
   │                        │                      │                        │
   │  VPC-A [VPC_CIDR_1]    │                      │  VPC-C [VPC_CIDR_3]    │
   │  (1x EC2 instance)     │                      │  (1x EC2 instance)     │
   └──────────────────────┘                      └──────────────────────┘
```

> 💡 This lab keeps it to **1 VPC per region with 1 EC2 instance each**, since the goal is to isolate and validate the **peering mechanism** itself. You can extend it with more VPCs per side using the same pattern as the base Transit Gateway lab.

### 8.3 Step 1 — Create TGW-1 in Region-1 (Console)

1. In the top-right region selector, switch to `[AWS_REGION_1]`
2. Navigate to **VPC Console → Transit Gateways → Transit Gateways → Create Transit Gateway**
3. **Name tag**: `TGW-1`
4. **Amazon side ASN**: `[TGW1_ASN]` (e.g. `64512`)
5. **Default route table association**: ✅ Enable
6. **Default route table propagation**: ✅ Enable
7. Click **Create Transit Gateway** → wait for state `Available`

If you don't already have a VPC attached in this region, create **VPC-A** (`[VPC_CIDR_1]`, e.g. `10.1.0.0/16`) with one subnet, then attach it to `TGW-1` via **Transit Gateway Attachments → Create Transit Gateway Attachment** (same steps as the base TGW lab).

### 8.4 Step 2 — Create TGW-2 in Region-2 (Console)

1. In the top-right region selector, switch to `[AWS_REGION_2]`
2. Navigate to **VPC Console → Transit Gateways → Transit Gateways → Create Transit Gateway**
3. **Name tag**: `TGW-2`
4. **Amazon side ASN**: `[TGW2_ASN]` (e.g. `64513` — must differ from TGW-1's ASN)
5. **Default route table association**: ✅ Enable
6. **Default route table propagation**: ✅ Enable
7. Click **Create Transit Gateway** → wait for state `Available`

Create **VPC-C** (`[VPC_CIDR_3]`, e.g. `10.3.0.0/16`) with one subnet in this region, then attach it to `TGW-2`.

> 📝 Note the **Transit Gateway ID** of `TGW-1` (e.g. `tgw-0111...`) before switching regions — you'll need to reference it when creating the peering request from Region-2 (or vice versa).

### 8.5 Step 3 — Create the Peering Attachment (Requester Side)

You can request peering from either side; this example requests from **Region-2 (TGW-2)** targeting **Region-1 (TGW-1)**.

1. While still in `[AWS_REGION_2]`, navigate to **VPC Console → Transit Gateways → Transit Gateway Attachments → Create Transit Gateway Attachment**
2. **Transit Gateway ID**: select `TGW-2`
3. **Attachment type**: `Peering Connection`
4. **Attachment name tag**: `TGW1-TGW2-Peering`
5. **Account**: `My account` (same-account peering for this lab — cross-account is covered in Section 9)
6. **Region**: select `[AWS_REGION_1]`
7. **Transit gateway (accepter)**: enter `TGW-1`'s Transit Gateway ID
8. Click **Create Transit Gateway Attachment**

The attachment will show state `Pending Acceptance`.

### 8.6 Step 4 — Accept the Peering Attachment (Accepter Side)

1. Switch the region selector to `[AWS_REGION_1]`
2. Navigate to **VPC Console → Transit Gateways → Transit Gateway Attachments**
3. You should see the peering attachment listed with state `Pending Acceptance`
4. Select it → **Actions → Accept Transit Gateway Peering Attachment**
5. Confirm. Wait for the state to change to `Available` (refresh periodically; can take a couple of minutes)

> ✅ Both sides now show the peering attachment as `Available` — the physical link exists, but **no traffic flows yet** until routes are added in Step 5.

### 8.7 Step 5 — Add Static Routes on Both TGW Route Tables (Console)

**On TGW-1's route table (Region-1):**

1. Navigate to **VPC Console → Transit Gateways → Transit Gateway Route Tables**
2. Select the route table associated with `TGW-1` (the default one, since propagation was enabled for local attachments)
3. **Routes** tab → **Create static route**
4. **CIDR**: `[VPC_CIDR_3]` (VPC-C's CIDR in Region-2, e.g. `10.3.0.0/16`)
5. **Attachment**: select the peering attachment (`TGW1-TGW2-Peering`)
6. Click **Create static route**

**On TGW-2's route table (Region-2):**

1. Switch region to `[AWS_REGION_2]`
2. Navigate to **VPC Console → Transit Gateways → Transit Gateway Route Tables**
3. Select the route table associated with `TGW-2`
4. **Routes** tab → **Create static route**
5. **CIDR**: `[VPC_CIDR_1]` (VPC-A's CIDR in Region-1, e.g. `10.1.0.0/16`)
6. **Attachment**: select the peering attachment
7. Click **Create static route**

> 🔑 This is the step beginners most often forget. The peering **attachment** being `Available` does NOT mean traffic flows — you must explicitly add the **static route** on each side pointing the remote CIDR at the peering attachment.

### 8.8 Step 6 — Update VPC Route Tables (Console)

**VPC-A's route table (Region-1):**
1. **VPC Console → Route Tables** → select VPC-A's route table → **Routes → Edit routes → Add route**
2. **Destination**: `[VPC_CIDR_3]` → **Target**: `Transit Gateway` → `TGW-1`
3. **Save changes**

**VPC-C's route table (Region-2):**
1. Switch region to `[AWS_REGION_2]`
2. **VPC Console → Route Tables** → select VPC-C's route table → **Routes → Edit routes → Add route**
3. **Destination**: `[VPC_CIDR_1]` → **Target**: `Transit Gateway` → `TGW-2`
4. **Save changes**

### 8.9 Step 7 — Security Groups (Console)

In **each region**, edit the security group attached to your test EC2 instance to allow ICMP/SSH from the **remote VPC's CIDR**:

- VPC-A's SG (Region-1): allow ICMP/SSH from `[VPC_CIDR_3]`
- VPC-C's SG (Region-2): allow ICMP/SSH from `[VPC_CIDR_1]`

### 8.10 Step 8 — Launch Test EC2 Instances (Console)

In each region, launch **1 EC2 instance** into the respective VPC/subnet (same console flow as the base Transit Gateway lab: **EC2 Console → Launch instances**, Amazon Linux 2023, `[EC2_INSTANCE_TYPE]`, no public IP, correct VPC/subnet/SG).

### 8.11 Step 9 — Validate Cross-Region Connectivity

1. Connect to the EC2 instance in **VPC-A** (Region-1) via EC2 Instance Connect or SSM Session Manager
2. From its terminal, ping the **private IP** of the instance in **VPC-C** (Region-2):

```bash
ping -c 4 [VPC_C_INSTANCE_PRIVATE_IP]
```

Expected result: successful replies, confirming traffic flows VPC-A → TGW-1 → Peering Attachment → TGW-2 → VPC-C, entirely over the AWS backbone.

### 8.12 Step 10 — Verify Routes in Both TGW Route Tables

1. **VPC Console → Transit Gateways → Transit Gateway Route Tables** (in each region)
2. Open the **Routes** tab for each TGW's route table
3. Confirm you see a **Static** route entry pointing the remote VPC CIDR to the peering attachment

---

## 9. Production-Level Architecture

### 9.1 Key Differences from the Lab

| Lab (Basic) | Production |
|---|---|
| Same-account peering | **Cross-account peering** common in multi-org/landing-zone setups (requester and accepter in different accounts) |
| Manual route acceptance via console | Automation via IaC (Terraform) with explicit accepter-side resources, or governed manual approval workflows |
| Static routes for 1–2 CIDRs | **Route summarization** (e.g., advertise a single summarized CIDR block per region instead of every individual VPC CIDR) to keep route tables manageable |
| One peering connection (2 regions) | **Full or partial mesh** of peering connections across 3+ regions, planned deliberately since peering is non-transitive |
| No monitoring | **TGW Flow Logs + CloudWatch** on both sides of every peering attachment |

### 9.2 Multi-Region Mesh Design (3+ Regions)

Because peering is non-transitive, connecting 3 regions requires a peering connection for **every pair**:

```
            Region A (TGW-1)
               /        \
       Peering /          \ Peering
             /              \
   Region B (TGW-2) ──Peering── Region C (TGW-3)

   3 regions = 3 peering connections (full mesh)
   N regions = N(N-1)/2 peering connections
```

For 4+ regions, evaluate whether a **hub region** pattern (one "core" TGW that every other region peers with) better fits the access pattern than a full mesh — but remember: the hub region's TGW does **not** automatically route Region-B traffic to Region-C just because both peer with it; you'd still need explicit routing logic, and AWS TGW peering itself remains point-to-point only. A hub-and-spoke region design is usually paired with a **centralized inspection/transit VPC** in the hub region rather than relying on "transit-like" behavior from TGW peering alone.

### 9.3 Cross-Account Peering Pattern

```
   Account: Network-Hub (Region-1)         Account: Spoke-Org (Region-2)
   ┌──────────────────────┐                ┌──────────────────────┐
   │   TGW-1                │   Peering      │   TGW-2                │
   │   (requester or         │◄──────────────►│   (accepter or          │
   │    accepter)            │                │    requester)           │
   └──────────────────────┘                └──────────────────────┘
```

Steps differ slightly from the same-account lab:
1. The **requester** creates the peering attachment, specifying the **accepter's AWS Account ID** and TGW ID
2. The **accepter account** must log into **its own console**, navigate to Transit Gateway Attachments, and accept the request (cross-account acceptance cannot be done from the requester's account)
3. Both account owners independently manage their own TGW route tables and static routes — there is no automatic route sharing across accounts

### 9.4 Route Summarization for Scalability

Instead of adding one static route per individual VPC CIDR across a peering connection, allocate region-level **summary CIDR blocks** in your IP address plan, e.g.:

```
Region-1 summary block: 10.1.0.0/14   (covers 10.1.0.0 - 10.4.255.255, i.e. VPCs 10.1.x - 10.4.x)
Region-2 summary block: 10.5.0.0/14   (covers VPCs 10.5.x - 10.8.x)
```

Then each side only needs **one static route** (the summary CIDR) pointing to the peering attachment, instead of one route per VPC — dramatically reducing route table maintenance as you add more VPCs.

### 9.5 Resilience Considerations

- TGW peering connections are inherently **highly available** within AWS's backbone (no single point of failure to manage on your side), but you should still monitor attachment health via CloudWatch
- For disaster recovery architectures, ensure **both** the primary and standby region's TGW route tables and VPC route tables are kept in sync via IaC — a manual route added during an incident is easy to forget to remove/replicate afterward
- Use **AWS Config rules** or scheduled Lambda checks to detect TGW route table drift between paired regions

### 9.6 Terraform Skeleton (Production Pattern)

```hcl
# Region 1 provider
provider "aws" {
  alias  = "region1"
  region = "[AWS_REGION_1]"
}

# Region 2 provider
provider "aws" {
  alias  = "region2"
  region = "[AWS_REGION_2]"
}

resource "aws_ec2_transit_gateway" "tgw1" {
  provider         = aws.region1
  description      = "TGW-1"
  amazon_side_asn  = 64512
  tags = { Name = "TGW-1" }
}

resource "aws_ec2_transit_gateway" "tgw2" {
  provider         = aws.region2
  description      = "TGW-2"
  amazon_side_asn  = 64513
  tags = { Name = "TGW-2" }
}

# Requester side (Region 2 requesting peering to Region 1)
resource "aws_ec2_transit_gateway_peering_attachment" "peering" {
  provider                = aws.region2
  transit_gateway_id      = aws_ec2_transit_gateway.tgw2.id
  peer_transit_gateway_id = aws_ec2_transit_gateway.tgw1.id
  peer_region             = "[AWS_REGION_1]"
  tags = { Name = "TGW1-TGW2-Peering" }
}

# Accepter side (Region 1 accepts the peering request)
resource "aws_ec2_transit_gateway_peering_attachment_accepter" "peering_accepter" {
  provider                      = aws.region1
  transit_gateway_attachment_id = aws_ec2_transit_gateway_peering_attachment.peering.id
  tags = { Name = "TGW1-TGW2-Peering-Accepter" }
}

# Static route on TGW-1's route table pointing to TGW-2's VPC CIDR
resource "aws_ec2_transit_gateway_route" "tgw1_to_region2" {
  provider                       = aws.region1
  destination_cidr_block         = var.vpc_c_cidr
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_peering_attachment_accepter.peering_accepter.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway.tgw1.association_default_route_table_id
}

# Static route on TGW-2's route table pointing to TGW-1's VPC CIDR
resource "aws_ec2_transit_gateway_route" "tgw2_to_region1" {
  provider                       = aws.region2
  destination_cidr_block         = var.vpc_a_cidr
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_peering_attachment.peering.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway.tgw2.association_default_route_table_id
}
```

---

## 10. Edge Cases

| Edge Case | What Happens | How to Handle |
|---|---|---|
| **Forgetting to accept the peering attachment** | Attachment stays `Pending Acceptance` indefinitely; no traffic flows | Accepter must explicitly accept via console (or Terraform accepter resource in IaC) |
| **Peering attachment Available, but no traffic flows** | Most common beginner mistake — attachment being "up" does not mean routes exist | Always add **static routes** on both TGW route tables pointing remote CIDRs to the peering attachment |
| **Expecting transitive routing through a 3rd TGW** | TGW-1 ↔ TGW-2 ↔ TGW-3 does NOT let TGW-1 reach TGW-3 | Add a direct TGW-1↔TGW-3 peering connection, or redesign with a centralized transit VPC pattern |
| **Overlapping CIDRs between regions** | Routing breaks/is ambiguous, same as standard TGW | Maintain a single global IP address plan (e.g., via IPAM) across all regions before creating VPCs |
| **Cross-account peering request not visible to accepter** | Accepter is logged into the wrong account, or peering request expired (requests can time out if not accepted within a window) | Confirm correct AWS Account ID was used as target; re-request if expired |
| **High inter-region data transfer costs discovered after rollout** | Often caused by an overly broad route (e.g., summarized CIDR routing more traffic than intended) or a chatty replication job | Audit CloudWatch data-processed metrics per attachment regularly; scope routes as narrowly as the use case allows |
| **ASN conflicts in advanced BGP scenarios** | Can cause issues if you later attach Direct Connect/VPN with overlapping ASN expectations | Assign distinct, deliberate ASNs per TGW from the start, documented in your network design |
| **Deleting a peering attachment that still has active routes** | Leaves black-holed static routes on both TGW route tables | Delete the static routes on **both** sides first, then delete the peering attachment |

---

## 11. Best Practices

- Always pair **peering attachment creation** with **explicit static route creation** on both sides as a single change set (in IaC, group them in the same apply/PR) to avoid "attachment up, but routes missing" gaps
- Use **distinct ASNs** per TGW from the start, even if not strictly required for basic peering, to avoid future BGP/Direct Connect conflicts
- Plan a **global, non-overlapping CIDR scheme** across all regions before any TGW or peering work begins (use AWS IPAM for large environments)
- Where possible, use **CIDR summarization** per region to keep static route counts low and maintainable as VPC count grows
- For cross-account peering, document **who owns acceptance** and embed it in your IaC pipeline (e.g., a manual approval gate) rather than relying on someone remembering to click "Accept" in the console
- Monitor **both sides** of every peering attachment with CloudWatch — a one-sided health check can miss accepter-side issues
- Avoid relying on TGW peering to simulate transitive multi-hop routing — if you need 3+ regions to freely talk to each other, explicitly design a full mesh or a dedicated transit/inspection region

---

## 12. Interview Q&A + Cheat Sheet

**Q1: Is Transit Gateway peering transitive?**
A: No. If TGW-1 is peered with TGW-2, and TGW-2 is peered with TGW-3, TGW-1 cannot reach TGW-3's VPCs unless a direct TGW-1↔TGW-3 peering connection also exists.

**Q2: Does an "Available" peering attachment mean traffic is flowing?**
A: No. The attachment being available only means the link exists. You must also add static (or propagated) routes in both TGW route tables, and routes in the VPC route tables, before traffic flows.

**Q3: Can you peer Transit Gateways across AWS accounts?**
A: Yes. The requester specifies the target account ID and TGW ID; the accepter must log into their own account to approve the peering attachment.

**Q4: Does TGW peering traffic go over the public internet?**
A: No. It travels over the AWS global backbone and is automatically encrypted in transit.

**Q5: How do you scale TGW peering to many regions without an unmanageable route table?**
A: Use CIDR summarization (allocate one summary block per region) so each peering connection only needs one or a few static routes instead of one per VPC.

**Q6: What's the difference between a VPC attachment's route propagation and a peering attachment's routing?**
A: VPC attachments commonly use automatic propagation into the TGW route table. Peering attachments are most often configured with explicit static routes (though propagation can also be enabled), making cross-region routing intentional and auditable.

**Cheat Sheet:**

```
Peering Attachment = Point-to-point link between exactly 2 TGWs
Requester          = Side that initiates the peering request
Accepter           = Side that must explicitly approve the request
Transitive?         = NO — peering never chains through a 3rd TGW
Routing             = Static routes (typical) on BOTH TGW route tables required
Encryption          = Automatic, over AWS global backbone
Cross-account       = Supported — accepter approves from their own account
Cross-region        = Most common use case (also supports same-region cross-account)
```

---

## 13. Cleanup

Tear down resources in this order using the **Console**, in **both** regions:

1. **Terminate test EC2 instances** (each region): **EC2 Console → Instances** → select → **Instance state → Terminate instance**
2. **Delete static routes** on both TGW route tables: **VPC Console → Transit Gateways → Transit Gateway Route Tables** → select route table → **Routes** tab → select the static route → **Delete static route**
3. **Delete the peering attachment**: from either region, **VPC Console → Transit Gateways → Transit Gateway Attachments** → select the peering attachment → **Actions → Delete Transit Gateway Attachment** (deleting from one side removes it from both)
4. **Delete VPC attachments** (each region): **Transit Gateway Attachments** → select the VPC attachment → **Actions → Delete Transit Gateway Attachment**
5. **Delete the Transit Gateways**: **Transit Gateways** → select `TGW-1` / `TGW-2` → **Actions → Delete Transit Gateway**
6. **Delete subnets, then VPCs** in each region: **VPC Console → Subnets → Delete subnet**, then **Your VPCs → Delete VPC**

> ⚠️ Delete static routes referencing the peering attachment **before** deleting the attachment itself, or you may end up with orphaned route entries that need manual cleanup.

---

**End of Guide.** Pair this with the base **AWS Transit Gateway** guide for a complete picture: single-region hub-and-spoke first, then multi-region/multi-account connectivity via peering on top of it.
