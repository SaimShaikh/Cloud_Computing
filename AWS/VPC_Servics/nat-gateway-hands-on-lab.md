# AWS NAT Gateway — Complete Hands-On Lab Guide

---

## Table of Contents

1. [Concept Deep Dive](#1-concept-deep-dive)
2. [NAT Gateway vs NAT Instance vs Internet Gateway](#2-nat-gateway-vs-nat-instance-vs-internet-gateway)
3. [Architecture Diagram](#3-architecture-diagram)
4. [Lab Objective](#4-lab-objective)
5. [Prerequisites](#5-prerequisites)
6. [Hands-On Lab — Step by Step (Console Only)](#6-hands-on-lab--step-by-step-console-only)
7. [Verification & Testing](#7-verification--testing)
8. [AWS CLI Reference (Equivalent Commands)](#8-aws-cli-reference-equivalent-commands)
9. [Terraform Code](#9-terraform-code)
10. [Edge Cases & Failure Scenarios](#10-edge-cases--failure-scenarios)
11. [Cost Considerations](#11-cost-considerations)
12. [Best Practices](#12-best-practices)
13. [Cleanup (Avoid Billing)](#13-cleanup-avoid-billing)
14. [Interview Q&A](#14-interview-qa)
15. [Cheat Sheet](#15-cheat-sheet)
16. [Mastery Checklist](#16-mastery-checklist)

---

## 1. Concept Deep Dive

### What is a NAT Gateway?

A **NAT (Network Address Translation) Gateway** is a managed AWS service that allows instances in a **private subnet** to initiate **outbound** connections to the internet (or other AWS services) — for things like downloading OS updates, pulling packages, calling external APIs — **without exposing those instances to inbound internet traffic**.

Think of it as a one-way door:
- Private EC2 instance → NAT Gateway → Internet Gateway → Internet ✅ (outbound allowed)
- Internet → NAT Gateway → Private EC2 instance ❌ (inbound blocked, NAT Gateway never initiates or forwards unsolicited inbound connections)

### Why does this matter?

In any real production VPC design, you never want your application servers, databases, or backend workloads to have a **public IP** or be directly reachable from the internet. But those instances often still need outbound internet access — e.g.:
- `apt-get update` / `yum update` on a private EC2 instance
- Pulling a Docker image from Docker Hub
- Calling a third-party payment API
- SSM Agent / CloudWatch Agent phoning home (though SSM VPC endpoints are the private-only alternative you already labbed)

NAT Gateway solves this by giving the **subnet** (via its route table) a path to the internet that:
- Only allows traffic that originated from inside the VPC
- Translates the private IP of the instance to the NAT Gateway's Elastic IP for outbound packets
- Tracks the connection so return traffic is routed back correctly
- Silently drops any unsolicited inbound connection attempts

### How does NAT actually work (packet-level)?

1. Private instance (10.0.2.10) wants to reach `8.8.8.8`.
2. Its route table says: "anything not in the VPC CIDR → send to NAT Gateway".
3. Traffic hits the NAT Gateway (sitting in a **public** subnet).
4. NAT Gateway rewrites the **source IP** from `10.0.2.10` → its own **Elastic IP** (e.g., `3.110.x.x`), and tracks this translation in a connection table (source port is also remapped).
5. NAT Gateway forwards the packet to the Internet Gateway → internet.
6. Response comes back to the NAT Gateway's Elastic IP.
7. NAT Gateway looks up its connection table, rewrites the **destination IP** back to `10.0.2.10`, and forwards it into the private subnet.

This is **Source NAT (SNAT)** for outbound-only, stateful, managed at the AWS hypervisor/network layer — you don't manage an OS, patch it, or scale it yourself.

### Key properties

| Property | Detail |
|---|---|
| Deployment location | Must live in a **public subnet** (subnet with a route to an Internet Gateway) |
| Direction | Outbound-initiated only; stateful return traffic allowed |
| IP requirement | Requires an **Elastic IP** attached at creation |
| Availability Zone scope | A NAT Gateway is zonal — it lives in one AZ. For HA, deploy one NAT Gateway per AZ |
| Bandwidth | Scales automatically up to 100 Gbps |
| Management | Fully managed by AWS — no patching, no scaling config, no failover config needed within its AZ |
| Pricing | Hourly charge + per-GB data processing charge (this is the #1 surprise cost in real AWS bills) |

---

## 2. NAT Gateway vs NAT Instance vs Internet Gateway

| Feature | Internet Gateway (IGW) | NAT Gateway | NAT Instance (self-managed EC2) |
|---|---|---|---|
| Purpose | Two-way internet access for **public** subnets | One-way (outbound) internet access for **private** subnets | Same as NAT Gateway, but manually run on EC2 |
| Direction | Inbound + Outbound | Outbound only (stateful return) | Outbound only (stateful return) |
| Managed by AWS | Yes (fully) | Yes (fully — patching, scaling, HA within AZ) | No — you patch OS, manage scaling, HA yourself |
| HA | N/A (regional, highly available by default) | Zonal — need 1 per AZ for multi-AZ HA | You build HA yourself (scripts, failover) |
| Bandwidth | N/A | Auto-scales to 100 Gbps | Capped by instance type |
| Cost model | Free | Hourly + per-GB processed | EC2 instance cost only (cheaper at small scale, but ops overhead) |
| Security Group | N/A | Cannot attach SG (uses NACLs only) | Can attach SG |
| Use today | Always, for any public subnet | Default recommended choice | Legacy / cost-sensitive / custom routing needs |

**Interview one-liner:** *"IGW gives two-way internet access to public subnets. NAT Gateway gives one-way (outbound-only) internet access to private subnets, and unlike a NAT instance, it's a managed, highly-scalable service that doesn't need patching or manual failover."*

---

## 3. Architecture Diagram

### Target lab architecture — single AZ, one public + one private subnet

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/5ef7d431-7ad1-40e0-bd4e-4cc9465cc4c9" />


                                  
### Packet flow for `curl https://checkip.amazonaws.com` from the private instance

```
private-instance (10.0.2.10)
      │  source IP: 10.0.2.10
      ▼
private-rt: 0.0.0.0/0 -> nat-gw-natlab
      ▼
NAT Gateway (in public subnet)
      │  SNAT: source IP rewritten to Elastic IP (3.110.x.x)
      ▼
public-rt: 0.0.0.0/0 -> igw-natlab
      ▼
Internet Gateway
      ▼
Internet (checkip.amazonaws.com)
      │  sees request from 3.110.x.x, NOT 10.0.2.10
      ▼
Response returns to 3.110.x.x
      ▼
NAT Gateway (connection table lookup) rewrites destination back to 10.0.2.10
      ▼
private-instance receives response
```

---

## 4. Lab Objective

By the end of this lab you will have:

1. Built a custom VPC with one public subnet and one private subnet in the same AZ.
2. Deployed an Internet Gateway and attached it to the VPC.
3. Deployed a NAT Gateway with an Elastic IP in the public subnet.
4. Configured route tables so the private subnet routes internet-bound traffic through the NAT Gateway.
5. Launched a bastion host (public) and a private EC2 instance (no public IP).
6. Proven that the private instance **can** reach the internet outbound (via NAT Gateway) but **cannot** be reached from the internet inbound.
7. Observed the NAT Gateway's Elastic IP as the source IP seen by the internet.
8. Cleaned up all billable resources.

---

## 5. Prerequisites

- AWS account with permissions for VPC, EC2, EIP.
- An existing EC2 Key Pair (or create a new one in this lab) — needed to SSH into the bastion host.
- Region used in this guide: **ap-south-1 (Mumbai)** — substitute your own region/AZ if different.
- Basic familiarity with SSH.
- No CLI needed for the hands-on steps below — everything is done in the **AWS Management Console**.

---

## 6. Hands-On Lab — Step by Step (Console Only)

### Step 1 — Create the VPC

1. Go to **VPC console** → **Your VPCs** → **Create VPC**.
2. Choose **VPC only** (not "VPC and more" — we want to build each piece manually so you understand every part).
3. Set:
   - **Name tag**: `nat-lab-vpc`
   - **IPv4 CIDR block**: `10.0.0.0/16`
   - **IPv6 CIDR block**: No IPv6
   - **Tenancy**: Default
4. Click **Create VPC**.

### Step 2 — Create the Public Subnet

1. Go to **Subnets** → **Create subnet**.
2. Select VPC: `nat-lab-vpc`.
3. Subnet settings:
   - **Subnet name**: `public-subnet-1a`
   - **Availability Zone**: `ap-south-1a`
   - **IPv4 CIDR block**: `10.0.1.0/24`
4. Click **Create subnet**.

### Step 3 — Create the Private Subnet

1. Still in **Subnets** → **Create subnet** (you can create multiple in the same wizard, or repeat).
2. Select VPC: `nat-lab-vpc`.
3. Subnet settings:
   - **Subnet name**: `private-subnet-1a`
   - **Availability Zone**: `ap-south-1a` (**same AZ** as the public subnet — this matters, NAT Gateway is zonal)
   - **IPv4 CIDR block**: `10.0.2.0/24`
4. Click **Create subnet**.

### Step 4 — Enable Auto-assign Public IP on the Public Subnet

1. Go to **Subnets** → select `public-subnet-1a`.
2. **Actions** → **Edit subnet settings**.
3. Check **Enable auto-assign public IPv4 address**.
4. Save.

> This ensures any instance you launch into the public subnet (like the bastion host) automatically gets a public IP. Do **NOT** do this for the private subnet — that's the whole point.

### Step 5 — Create and Attach the Internet Gateway

1. Go to **Internet Gateways** → **Create internet gateway**.
2. Name: `igw-natlab`.
3. Click **Create internet gateway**.
4. Select it → **Actions** → **Attach to VPC** → choose `nat-lab-vpc` → **Attach internet gateway**.

### Step 6 — Allocate an Elastic IP (for the NAT Gateway)

1. Go to **Elastic IPs** → **Allocate Elastic IP address**.
2. Network Border Group: leave default.
3. Click **Allocate**.
4. (Optional) Rename tag to `eip-natgw`.

> You must do this **before or during** NAT Gateway creation — a NAT Gateway cannot exist without an Elastic IP attached.

### Step 7 — Create the Public Route Table

1. Go to **Route Tables** → **Create route table**.
2. Name: `public-rt`.
3. VPC: `nat-lab-vpc`.
4. Click **Create route table**.
5. Select `public-rt` → **Routes** tab → **Edit routes** → **Add route**:
   - Destination: `0.0.0.0/0`
   - Target: **Internet Gateway** → select `igw-natlab`
6. Save routes.
7. Go to **Subnet associations** tab → **Edit subnet associations** → check `public-subnet-1a` → **Save associations**.

<img width="2813" height="867" alt="image" src="https://github.com/user-attachments/assets/4a1d4e84-ca0d-462e-bb95-8c591cdcf2a6" />

<img width="2805" height="841" alt="image" src="https://github.com/user-attachments/assets/8c7e9e66-a1f6-43ac-98a3-378f56ee2eaf" />


### Step 8 — Create the NAT Gateway

1. Go to **VPC console** → **NAT Gateways** → **Create NAT gateway**.
2. Settings:
   - **Name**: `nat-gw-natlab`
   - **Subnet**: `public-subnet-1a` (**critical** — NAT Gateway must sit in the public subnet, not the private one)
   - **Connectivity type**: **Public**
   - **Elastic IP allocation ID**: select the EIP you allocated in Step 7 (`eip-natgw`)
3. Click **Create NAT gateway**.
4. Wait for **State** to change from `Pending` → `Available` (usually takes 1–3 minutes).

<img width="1408" height="316" alt="Screenshot 2026-07-07 at 5 51 23 PM" src="https://github.com/user-attachments/assets/852dbd80-ae75-471a-ad25-9363dbaf9853" />

### Step 9 — Create the Private Route Table

1. Go to **Route Tables** → **Create route table**.
2. Name: `private-rt`.
3. VPC: `nat-lab-vpc`.
4. Click **Create route table**.
5. Select `private-rt` → **Routes** tab → **Edit routes** → **Add route**:
   - Destination: `0.0.0.0/0`
   - Target: **NAT Gateway** → select `nat-gw-natlab`
6. Save routes.
7. Go to **Subnet associations** tab → **Edit subnet associations** → check `private-subnet-1a` → **Save associations**.

<img width="2810" height="1007" alt="image" src="https://github.com/user-attachments/assets/0aabadf4-6ea5-4a84-8bb0-6f521146a63d" />

<img width="2793" height="894" alt="image" src="https://github.com/user-attachments/assets/a672b8ae-dca9-4e7e-b828-157ee76308ce" />


> **Common mistake to avoid**: If you accidentally point `private-rt`'s `0.0.0.0/0` route to the Internet Gateway instead of the NAT Gateway, your "private" instance would need a public IP to get internet access, defeating the entire purpose of this lab. Double-check the target says **NAT Gateway**, not **Internet Gateway**.

### Step 10 — Create a Security Group for the Bastion Host

1. Go to **Security Groups** → **Create security group**.
2. Name: `bastion-sg`.
3. VPC: `nat-lab-vpc`.
4. Inbound rules:
   - Type: SSH, Port 22, Source: **My IP** (never use 0.0.0.0/0 for SSH in real use)
5. Outbound rules: leave default (all traffic allowed).
6. Create security group.

<img width="2811" height="1191" alt="Screenshot 2026-07-07 at 5 55 08 PM" src="https://github.com/user-attachments/assets/e86c93b9-0ce9-4b20-997b-cbae1b696242" />

### Step 11 — Create a Security Group for the Private Instance

1. Go to **Security Groups** → **Create security group**.
2. Name: `private-sg`.
3. VPC: `nat-lab-vpc`.
4. Inbound rules:
   - Type: SSH, Port 22, Source: **Custom** → select `bastion-sg` (so only the bastion can SSH in, nothing from the open internet)
5. Outbound rules: leave default (all traffic allowed — needed for the NAT Gateway test).
6. Create security group.

<img width="3276" height="774" alt="image" src="https://github.com/user-attachments/assets/683e1937-51b3-4474-b788-3825aeb067c3" />


### Step 12 — Launch the Bastion Host (Public Subnet)

1. Go to **EC2 console** → **Launch instance**.
2. Name: `bastion-host`.
3. AMI: **Amazon Linux 2023**.
4. Instance type: `t2.micro` (free tier eligible).
5. Key pair: select an existing key pair, or create a new one and download the `.pem` file.
6. Network settings → **Edit**:
   - VPC: `nat-lab-vpc`
   - Subnet: `public-subnet-1a`
   - Auto-assign public IP: **Enable**
   - Security group: select existing → `bastion-sg`
7. Launch instance.

### Step 13 — Launch the Private Instance (Private Subnet)

1. Go to **EC2 console** → **Launch instance**.
2. Name: `private-instance`.
3. AMI: **Amazon Linux 2023**.
4. Instance type: `t2.micro`.
5. Key pair: use the **same key pair** as the bastion (simplifies this lab — in production you'd use separate credentials or SSM).
6. Network settings → **Edit**:
   - VPC: `nat-lab-vpc`
   - Subnet: `private-subnet-1a`
   - Auto-assign public IP: **Disable** (this must be disabled — confirm it explicitly, don't just leave the subnet default)
   - Security group: select existing → `private-sg`
7. Launch instance.
8. Wait for both instances to reach **Running** state and pass status checks.

---

## 7. Verification & Testing

### Test 1 — Confirm the private instance has NO public IP

1. Go to **EC2 console** → select `private-instance`.
2. Check the **Details** tab: **Public IPv4 address** should be **empty/blank**. This confirms it's unreachable directly from the internet.

### Test 2 — SSH into the bastion host

From your local machine:

```bash
chmod 400 your-key.pem
ssh -i your-key.pem ec2-user@<bastion-public-ip>
```

### Test 3 — Hop from bastion into the private instance

You need the private instance's **private IP** (e.g., `10.0.2.10`, visible in the EC2 console **Details** tab).

From the **bastion host**, you first need the private key available there too (copy it securely, or better, use SSH agent forwarding):

```bash
# On your local machine — copy the key to the bastion (one-time, for lab purposes only)
scp -i your-key.pem your-key.pem ec2-user@<bastion-public-ip>:~/

# Back on the bastion host
chmod 400 your-key.pem
ssh -i your-key.pem ec2-user@10.0.2.10
```

You are now logged into the **private instance**, which has no direct route from the internet.

### Test 4 — Prove outbound internet access works (via NAT Gateway)

On the **private instance**:

```bash
curl https://saimeshaikh.in
```

**Expected result:** This returns an IP address — but it will be the **NAT Gateway's Elastic IP** (e.g., `3.110.x.x`), **not** the private instance's own IP (`10.0.2.10`). This is the SNAT translation happening in real time.

Also try:

```bash
ping -c 4 8.8.8.8
sudo apt-get update -y
```

Both should succeed — proving outbound internet access works even though the instance has no public IP.

### Test 5 — Prove inbound internet access does NOT work

From your **local machine** (not the bastion), try to SSH directly to the private instance's private IP — this will simply hang/fail, since `10.0.2.0/24` is not routable from the public internet at all, and even if it were, the NAT Gateway does not forward unsolicited inbound connections.

```bash
ssh -i your-key.pem ec2-user@10.0.2.10
# Expected: times out / no route — private IPs aren't internet-routable and NAT Gateway blocks unsolicited inbound anyway
```

### Test 6 — Watch the NAT Gateway metrics

1. Go to **VPC console** → **NAT Gateways** → select `nat-gw-natlab`.
2. **Monitoring** tab → observe `BytesOutToDestination` / `BytesOutToSource` / `ActiveConnectionCount` ticking up right after you ran the `curl` and `ping` commands.

<img width="2789" height="1623" alt="image" src="https://github.com/user-attachments/assets/7780a262-6410-45ad-a0b3-b4690dfb740d" />

---




## 10. Edge Cases & Failure Scenarios

| Scenario | What happens | Root cause | Fix |
|---|---|---|---|
| Private instance can't reach internet at all | `curl` times out | Private route table `0.0.0.0/0` is missing, or points to IGW instead of NAT GW, or NAT Gateway subnet association is wrong | Verify `private-rt` target for `0.0.0.0/0` is the NAT Gateway, not IGW |
| NAT Gateway stuck in "Pending" | Takes longer than a few minutes | Rare AWS-side delay, or subnet/EIP misconfiguration | Wait 5 min; if still pending, delete and recreate |
| NAT Gateway in wrong AZ | Private subnet in `ap-south-1b` routes to NAT Gateway in `ap-south-1a` — traffic still works but crosses AZ boundary | NAT Gateway is zonal; cross-AZ routing works but is **not resilient** and adds cross-AZ data transfer cost | Deploy one NAT Gateway per AZ for HA and to avoid cross-AZ charges |
| Single point of failure | If the one AZ's NAT Gateway fails, every private subnet routed to it loses internet access | Only one NAT Gateway deployed for a multi-AZ VPC | Deploy NAT Gateway per AZ, each private subnet's route table points to the NAT Gateway in its **own** AZ |
| Unexpected high bill | NAT Gateway data processing charges are high (per-GB) | Large data transfers routed through NAT Gateway unnecessarily (e.g., S3/DynamoDB traffic that could use a Gateway VPC Endpoint instead) | Use **VPC Gateway Endpoints** for S3/DynamoDB to bypass NAT Gateway entirely for that traffic |
| Security Group confusion | Trying to attach a Security Group directly to the NAT Gateway | NAT Gateway does not support Security Groups | Control traffic via NACLs on the subnet, or SGs on the instances behind it |
| "Can't SSH to private instance" from your laptop directly | Connection just hangs, no error | Private subnet has no route to/from internet at all (correct, expected behavior) | Use bastion host / Session Manager (SSM) as the jump point, as done in this lab |
| NAT Gateway shows "Available" but Elastic IP release fails during cleanup | `Address is in use` error | NAT Gateway not fully deleted yet — deletion is asynchronous, takes a couple minutes | Wait for NAT Gateway state to become `Deleted`, then release the EIP |
| Route table not associated with the intended subnet | Instance behaves like it's using the main/default route table | Forgot to explicitly associate `private-rt` with `private-subnet-1a` | Always verify subnet associations tab after creating a custom route table |

---

## 11. Cost Considerations

- **NAT Gateway hourly charge**: billed per hour, per NAT Gateway, whether or not traffic passes through it.
- **Data processing charge**: billed per GB processed through the NAT Gateway — this is separate from, and in addition to, standard data transfer charges.
- **Multi-AZ HA doubles/triples the hourly cost**: one NAT Gateway per AZ means paying the hourly charge multiple times.
- **Cheaper alternative for dev/test**: a single NAT Gateway (or even a NAT instance on `t3.nano` for very low-traffic labs) if HA isn't required.
- **Reduce NAT Gateway data charges**: route S3/DynamoDB traffic through **VPC Gateway Endpoints** instead of letting it go through the NAT Gateway.

---

## 12. Best Practices

1. **One NAT Gateway per Availability Zone** in production for high availability — never rely on a single NAT Gateway across multiple AZs for a critical workload.
2. **Route each private subnet to the NAT Gateway in its own AZ** — avoids cross-AZ data transfer charges and avoids single-AZ dependency.
3. **Use VPC Gateway Endpoints for S3 and DynamoDB** to avoid unnecessary NAT Gateway data processing charges for traffic to those services.
4. **Never put a NAT Gateway in a private subnet** — it must sit in a public subnet with a route to an Internet Gateway.
5. **Tag everything** (`Name`, `Environment`, `Project`) — makes cost allocation and cleanup far easier, especially across multiple labs.
6. **Monitor NAT Gateway CloudWatch metrics** (`BytesOutToDestination`, `PacketsDropCount`, `ErrorPortAllocation`) in production to catch port exhaustion or throughput issues early.
7. **Consider SSM Session Manager** instead of a bastion host + SSH for private instance access — no bastion to patch, no SSH keys to manage, no inbound port 22 anywhere (as in your earlier SSM Session Manager lab).
8. **Delete idle NAT Gateways** in non-production environments when not in active use — the hourly charge continues to accrue even with zero traffic.

---

## 13. Cleanup (Avoid Billing)

Do this in order — **NAT Gateway and Elastic IP are billed even when idle**, so don't skip cleanup.

1. **Terminate EC2 instances**: `bastion-host` and `private-instance`.
2. **Delete the NAT Gateway**: VPC console → NAT Gateways → select `nat-gw-natlab` → **Actions** → **Delete NAT gateway**. Wait for state to become `Deleted` (takes a few minutes — it's asynchronous).
3. **Release the Elastic IP**: Elastic IPs → select `eip-natgw` → **Actions** → **Release Elastic IP addresses** (only works after the NAT Gateway is fully deleted).
4. **Delete route tables**: `public-rt` and `private-rt` (disassociate subnets first if the console requires it).
5. **Detach and delete the Internet Gateway**: select `igw-natlab` → **Actions** → **Detach from VPC**, then **Actions** → **Delete internet gateway**.
6. **Delete subnets**: `public-subnet-1a`, `private-subnet-1a`.
7. **Delete security groups**: `bastion-sg`, `private-sg` (delete `private-sg` first since it references `bastion-sg`).
8. **Delete the VPC**: `nat-lab-vpc`.
9. Double check **Elastic IPs** page is empty — an unattached Elastic IP left behind is a common silent billing leak.

---

## 14. Interview Q&A

**Q1: What is a NAT Gateway and why is it needed?**
A: It's a managed AWS service that lets instances in a private subnet initiate outbound internet connections while remaining unreachable from inbound internet traffic. It's needed because production workloads (app servers, DBs) shouldn't have public IPs, but often still need outbound access for updates, APIs, etc.

**Q2: Where must a NAT Gateway be deployed?**
A: In a **public subnet** — one that has a route to an Internet Gateway. It cannot function from a private subnet.

**Q3: Is NAT Gateway regional or zonal?**
A: Zonal. A NAT Gateway lives in a single Availability Zone. For multi-AZ high availability, you deploy one NAT Gateway per AZ, each with its own Elastic IP.

**Q4: Can you attach a Security Group to a NAT Gateway?**
A: No. NAT Gateways don't support Security Groups. Traffic control at that layer is done through subnet **Network ACLs**, and Security Groups on the actual EC2 instances.

**Q5: NAT Gateway vs NAT Instance — when would you choose the instance?**
A: NAT instance is self-managed EC2 — cheaper at very low/predictable traffic, and you can attach Security Groups or do custom packet processing, but you own patching, scaling, and HA. NAT Gateway is fully managed, auto-scales to 100 Gbps, but costs more and has less flexibility. Modern default is NAT Gateway unless there's a specific cost or customization reason not to.

**Q6: How does NAT Gateway achieve the "outbound only" behavior?**
A: It performs stateful Source NAT (SNAT). It tracks connections it initiated from inside the VPC in a connection table; only return traffic matching an existing tracked connection is allowed back in. It doesn't forward unsolicited inbound connections at all.

**Q7: What's the most common way people overspend on NAT Gateway?**
A: Sending high-volume traffic to S3 or DynamoDB through the NAT Gateway instead of using a **VPC Gateway Endpoint**, which routes that traffic privately within AWS's network at no NAT Gateway data-processing cost.

**Q8: What happens if the NAT Gateway's Availability Zone goes down?**
A: All private subnets whose route tables point to that NAT Gateway lose outbound internet access, since NAT Gateway has no automatic cross-AZ failover. This is why production designs use one NAT Gateway per AZ.

**Q9: Does a private subnet with a NAT Gateway route become "public"?**
A: No. A subnet is only "public" if its route table sends `0.0.0.0/0` directly to an **Internet Gateway**. Routing to a NAT Gateway keeps it private — instances still have no public IP and can't be reached inbound from the internet.

**Q10: What IP does an external server see when your private EC2 instance calls out to it?**
A: The NAT Gateway's Elastic IP — not the private instance's private IP. This is exactly what you verified in the lab using `curl https://checkip.amazonaws.com`.

---

## 15. Cheat Sheet

```
NAT Gateway = outbound-only internet access for PRIVATE subnets
Lives in    = PUBLIC subnet (needs route to IGW)
Needs       = Elastic IP (mandatory, attached at creation)
Scope       = Zonal (one AZ) -> 1 NAT GW per AZ for HA
SG support  = NO (use NACLs / instance-level SGs instead)
Billing     = Hourly rate + per-GB data processing (idle still billed)
Route rule  = private-rt: 0.0.0.0/0 -> nat-gw-id
Public rule = public-rt:  0.0.0.0/0 -> igw-id
Bypass tip  = Use VPC Gateway Endpoints for S3/DynamoDB to cut NAT costs
Access test = curl https://checkip.amazonaws.com from private instance
              -> should show NAT Gateway's EIP, not the instance's private IP
Cleanup ord = EC2 -> NAT GW -> EIP -> Route Tables -> IGW -> Subnets -> SGs -> VPC
```

---

## 16. Mastery Checklist

- [ ] I can explain what a NAT Gateway does and why private subnets need it.
- [ ] I know NAT Gateway must be deployed in a public subnet, never a private one.
- [ ] I understand NAT Gateway is zonal, not regional, and why that requires 1-per-AZ for HA.
- [ ] I built the VPC, both subnets, IGW, NAT Gateway, and both route tables manually via console.
- [ ] I correctly pointed `private-rt`'s `0.0.0.0/0` route to the NAT Gateway (not the IGW).
- [ ] I launched a bastion host and a private instance with no public IP.
- [ ] I proved outbound access works from the private instance (`curl`, `ping`, `dnf update`).
- [ ] I proved the NAT Gateway's Elastic IP — not the instance's own IP — is what the internet sees.
- [ ] I proved inbound access to the private instance from the open internet does not work.
- [ ] I checked NAT Gateway CloudWatch metrics during my test traffic.
- [ ] I understand the cost model (hourly + per-GB) and how VPC Gateway Endpoints reduce it for S3/DynamoDB.
- [ ] I know NAT Gateway does not support Security Groups.
- [ ] I completed full cleanup and confirmed no lingering Elastic IP or NAT Gateway charges.
- [ ] I can write the Terraform equivalent of this entire lab from memory.
- [ ] I can answer all 10 interview questions above without looking.

---

*End of guide. Next logical labs to pair with this: VPC Gateway Endpoints (S3/DynamoDB) to see the cost-saving alternative in action, and multi-AZ NAT Gateway HA setup.*
