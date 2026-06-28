# Amazon EC2 Replication with Same Network Configuration
## Complete Production Guide — Including Live Traffic Architecture

> **Who this is for:** DevOps / Cloud Engineers (0–3 years) preparing for production work and AWS interviews.
> **Covers:** EC2 networking internals, all replication methods, end-to-end production traffic architecture, hands-on lab, CLI, Terraform, troubleshooting, and 30+ interview questions.

---

# Table of Contents

1. [Introduction](#1-introduction)
2. [Learning Objectives](#2-learning-objectives)
3. [AWS Networking Fundamentals](#3-aws-networking-fundamentals)
4. [Understanding ENI](#4-understanding-eni)
5. [How EC2 Gets an IP](#5-how-ec2-gets-an-ip)
6. [Why Same IP Cannot Exist (on Two Running Instances)](#6-why-same-ip-cannot-exist-on-two-running-instances)
7. [Ways to Replicate EC2](#7-ways-to-replicate-ec2)
8. [Production Decision Matrix](#8-production-decision-matrix)
9. [End-to-End Production Traffic Architecture](#9-end-to-end-production-traffic-architecture)
10. [Architecture Diagrams](#10-architecture-diagrams)
11. [Hands-on Lab](#11-hands-on-lab)
12. [AWS CLI Reference](#12-aws-cli-reference)
13. [Terraform Version](#13-terraform-version)
14. [Edge Cases](#14-edge-cases)
15. [Assumptions](#15-assumptions)
16. [Failure Scenarios](#16-failure-scenarios)
17. [Best Practices](#17-best-practices)
18. [Security Deep Dive](#18-security-deep-dive)
19. [Troubleshooting Guide](#19-troubleshooting-guide)
20. [Interview Questions](#20-interview-questions)
21. [Summary and Cheat Sheet](#21-summary-and-cheat-sheet)
22. [Mastery Checklist](#22-mastery-checklist)

---

# 1. Introduction

## What is EC2 Replication?

EC2 replication means creating a new EC2 instance that is functionally identical to an existing one — same operating system, same software, same configuration, and as much of the same network identity as AWS permits.

Replication is NOT the same as:
- **Backup** — a backup stores data; a replicated instance is a running compute resource
- **Snapshot** — a snapshot captures EBS state at a point in time; replication produces a usable instance
- **Migration** — migration moves a workload; replication creates a parallel copy

## Why Organizations Clone EC2 Instances

| Reason | Scenario |
|---|---|
| Disaster Recovery (DR) | Primary instance fails; replica takes over |
| High Availability (HA) | Load Balancer distributes traffic to multiple identical instances |
| Blue/Green Deployments | New version launched as clone; traffic switched atomically |
| Horizontal Scaling | Auto Scaling Group launches clones under load |
| Environment Parity | Dev/staging cloned from production AMI |
| Forensic Investigation | Clone instance before making changes to a compromised host |

## Common Misconceptions

**Misconception 1: "I can copy the exact same private IP to a new instance"**
You cannot have two running instances with the same private IP in the same VPC at the same time. AWS prevents this at the hypervisor level. See Chapter 6.

**Misconception 2: "Copying the AMI copies the IP"**
An AMI captures the OS disk only. IP addresses are assigned by the VPC DHCP server at launch, not stored in the AMI.

**Misconception 3: "Stopping and starting gives the same public IP"**
Stop → Start releases the auto-assigned public IP and assigns a new one. Only Elastic IPs persist across stop/start.

**Misconception 4: "ENI can be used on any instance"**
ENIs are AZ-scoped. You cannot attach an ENI from `ap-south-1a` to an instance in `ap-south-1b`.

## AWS Limitations (Hard Constraints)

- One primary private IP per ENI (secondary private IPs are supported)
- Two running instances cannot share the same private IP in the same VPC
- ENIs cannot be moved across AZs
- ENIs cannot be moved across VPCs
- Public IPs (auto-assigned) are released on stop; Elastic IPs are not
- MAC address is bound to ENI, not to the instance

## Supported vs Unsupported Methods

| Goal | Supported | Not Supported |
|---|---|---|
| Same OS / software | AMI-based launch | — |
| Same private IP | ENI reuse OR release + reuse | Concurrent duplicate IP |
| Same public IP | Elastic IP | Auto-assigned public IP |
| Same MAC address | ENI reuse | New ENI with same MAC |
| Same security groups | Specify at launch | — |
| Same IAM role | Specify at launch | — |

---

# 2. Learning Objectives

By the end of this guide you will be able to:

**Networking**
- Explain VPC, Subnet, CIDR, ENI, and IP allocation from first principles
- Describe how DHCP assigns IPs during EC2 launch
- Explain why duplicate private IPs are impossible and how ARP prevents them
- Design a multi-AZ VPC with public and private subnets

**EC2 Operations**
- Create an AMI (with and without reboot) and explain the tradeoffs
- Launch an instance that replicates all network configuration of an existing instance
- Move an Elastic IP from one instance to another with zero manual DNS changes
- Reuse a private IP after the original instance has released it

**Production Architecture**
- Design an end-to-end production traffic architecture for an EC2-based service
- Explain the role of Route 53, WAF, CloudFront, ALB, and Auto Scaling Group
- Configure an ALB target group with health checks
- Implement a NAT Gateway for private subnet internet access
- Connect EC2 to RDS, ElastiCache, and S3 via VPC endpoints

**Security**
- Differentiate Security Groups (stateful) from Network ACLs (stateless)
- Implement least-privilege IAM roles for EC2 instances
- Enable VPC Flow Logs, CloudTrail, and GuardDuty
- Replace SSH/bastion with SSM Session Manager

**Automation**
- Use AWS CLI to describe, clone, and reconfigure instances
- Write Terraform to reproduce the full architecture as code

**Interviews**
- Answer 30+ scenario-based questions at Solutions Architect / DevOps Engineer level

---

# 3. AWS Networking Fundamentals

## VPC (Virtual Private Cloud)

A VPC is a logically isolated section of the AWS cloud where you define your own network. Think of it as your private data center inside AWS.

```
AWS Account
│
└── VPC: 10.0.0.0/16  (65,536 IPs)
    │
    ├── Public Subnet:  10.0.1.0/24  (256 IPs, AZ-a)
    ├── Public Subnet:  10.0.2.0/24  (256 IPs, AZ-b)
    ├── Private Subnet: 10.0.10.0/24 (256 IPs, AZ-a)
    └── Private Subnet: 10.0.11.0/24 (256 IPs, AZ-b)
```

Key VPC properties:
- Each VPC gets one primary CIDR block (e.g., `10.0.0.0/16`)
- One VPC can span multiple AZs but NOT multiple regions
- VPCs are region-scoped; subnets are AZ-scoped
- AWS reserves 5 IPs in every subnet (first 4 + last 1)

## Subnet

A subnet is a range of IPs inside a VPC, tied to exactly one Availability Zone.

**Public subnet:** Has a route to an Internet Gateway. Instances can receive public IPs.

**Private subnet:** No direct route to the internet. Instances use a NAT Gateway for outbound access.

**Reserved IPs example** (for `10.0.1.0/24`):
| IP | AWS use |
|---|---|
| 10.0.1.0 | Network address |
| 10.0.1.1 | VPC router |
| 10.0.1.2 | DNS server |
| 10.0.1.3 | Reserved for future use |
| 10.0.1.255 | Broadcast address (not used) |

**Usable IPs:** 256 - 5 = 251

## CIDR (Classless Inter-Domain Routing)

CIDR notation expresses an IP range as `base_address/prefix_length`.

```
10.0.0.0/16  →  10.0.0.0  to  10.0.255.255  (65,536 IPs)
10.0.1.0/24  →  10.0.1.0  to  10.0.1.255    (256 IPs)
10.0.1.0/28  →  10.0.1.0  to  10.0.1.15     (16 IPs)
```

Rule: the smaller the prefix number, the larger the range. `/16` is bigger than `/24`.

## ENI (Elastic Network Interface)

An ENI is a virtual network card. Every EC2 instance has at least one ENI (eth0). The ENI is the object that holds:
- Primary private IP
- Secondary private IPs (optional)
- Public IP association
- Elastic IP association
- Security groups
- MAC address
- Source/Destination Check flag

**Critical:** The ENI exists independently from the instance. You can detach it and attach it to another instance. The MAC address travels with the ENI.

## Primary ENI vs Secondary ENI

| Property | Primary ENI (eth0) | Secondary ENI (eth1+) |
|---|---|---|
| Created at launch | Always | Optional |
| Detachable while running | No | Yes |
| Deleted on instance termination | Yes (default) | Configurable |
| Use case | Standard network | Multi-homed instances, license tied to MAC |

## Private IP

A private IP is the instance's address inside the VPC. It is:
- Assigned from the subnet's CIDR range
- Persistent across stop/start (as long as the instance is not terminated)
- Not routable on the internet
- Bound to the ENI, not the instance directly

## Public IP (Auto-assigned)

- Assigned from AWS's pool when instance launches (if subnet setting enables it)
- Released when instance is stopped or terminated
- **Cannot be controlled or predicted** — changes on every stop/start
- Cannot be moved to another instance

## Elastic IP (EIP)

- A static public IP that you own until you explicitly release it
- Persists across stop/start
- Can be moved between instances (disassociate → associate)
- One free EIP per running instance; charged when unattached
- Backed by AWS Bring Your Own IP (BYOIP) or AWS-owned pool

## MAC Address

- Every ENI gets a unique MAC address at creation
- The MAC address never changes for the life of the ENI
- Software licensing systems often bind to MAC address — this is why ENI reuse is important for license continuity

## DNS

AWS provides two DNS options within a VPC:
- **Private DNS:** `ip-10-0-1-10.ap-south-1.compute.internal` (auto-generated from private IP)
- **Public DNS:** `ec2-13-235-100-50.ap-south-1.compute.amazonaws.com` (only when public IP assigned)

Enabling `enableDnsHostnames` and `enableDnsSupport` on the VPC is required for DNS to work.

## Source/Destination Check

By default, AWS checks that all traffic sent or received by an ENI has a source or destination IP that matches the ENI's own IP. If it doesn't match, the packet is dropped.

**When to disable:** NAT instances and VPN/proxy appliances need to forward traffic on behalf of others. Disable Source/Destination Check on those ENIs.

---

# 4. Understanding ENI

## Complete ENI Architecture

```
                    EC2 Instance
                 ┌──────────────────────────────┐
                 │                              │
                 │    Application / OS          │
                 │    (eth0, eth1 visible)       │
                 │                              │
                 └──────────┬───────────────────┘
                            │
                  ┌─────────▼──────────┐
                  │   Primary ENI      │  ← eth0 (eni-0a1b2c3d)
                  │                    │
                  ├────────────────────┤
                  │ Primary Private IP │  10.0.1.10
                  │ Secondary Pvt IPs  │  10.0.1.11, 10.0.1.12 (optional)
                  │ Public IP          │  54.123.45.67 (if assigned)
                  │ Elastic IP         │  13.235.100.50 (if attached)
                  │ Security Groups    │  sg-app, sg-ssh
                  │ MAC Address        │  02:42:ac:11:00:02
                  │ Source/Dest Check  │  Enabled (default)
                  │ Subnet             │  subnet-0a1b2c3d (10.0.1.0/24)
                  │ AZ                 │  ap-south-1a
                  └────────────────────┘
```

## Every Field Explained

**ENI ID (`eni-xxxxxxxxxx`):** Unique identifier for this network interface. Use this when you want to move the ENI, check its properties, or tag it.

**Primary Private IP:** The main IP address of this interface within the subnet. Assigned at creation, stays for the life of the instance (unless terminated).

**Secondary Private IPs:** Additional private IPs on the same interface. Useful for hosting multiple SSL certs, running multiple services each needing their own IP, or IP aliasing.

**Public IP:** Auto-assigned from AWS pool. Lost on stop. Not moveable.

**Elastic IP:** Static public IP you own. Survives stop/start. Moveable across instances.

**Security Groups:** Stateful firewall rules attached to the ENI (not the subnet). Multiple SGs can be attached; rules are unioned.

**MAC Address:** Hardware-level identifier. Fixed to this ENI. Used by software licensing. When you move an ENI, the MAC travels with it.

**Source/Destination Check:** AWS's anti-spoofing mechanism. Disable for NAT/VPN appliances.

**Subnet:** The ENI is pinned to one subnet, which means it's pinned to one AZ. Cannot cross AZ boundaries.

---

# 5. How EC2 Gets an IP

## Full Lifecycle: Launch Request to Network-Ready

```
Step 1: You submit RunInstances API call
        (specifying subnet, SG, AMI, etc.)
        │
Step 2: AWS selects a free IP from the subnet's CIDR pool
        (or uses the IP you specified with --private-ip-address)
        │
Step 3: AWS creates an ENI in the subnet
        Assigns: primary private IP, MAC address, SG rules
        │
Step 4: AWS hypervisor connects the ENI to the physical NIC
        VPC router is updated: 10.0.1.10 → this ENI
        │
Step 5: Instance boots, OS runs DHCP DISCOVER
        │
Step 6: AWS DHCP server (at VPC router 10.0.0.2) responds
        Sends: IP, subnet mask, gateway, DNS
        DHCP lease time: effectively infinite (renewed automatically)
        │
Step 7: OS configures eth0 with received IP
        Instance is now reachable within VPC
        │
Step 8 (if public IP requested):
        AWS maps a public IP from its pool → ENI's private IP
        This mapping lives in the Internet Gateway (NAT at IGW level)
        The OS never sees the public IP — only the private IP
```

## DHCP in AWS

AWS runs a DHCP server at the `.2` address of every VPC CIDR (e.g., `10.0.0.2`). When an instance boots:

1. OS sends `DHCP DISCOVER` (broadcast)
2. AWS DHCP responds with `DHCP OFFER` (IP, mask, gateway, DNS, lease)
3. OS accepts with `DHCP REQUEST`
4. AWS confirms with `DHCP ACK`

The lease is long enough that you never see an IP change while an instance is running. However, on **stop → start**, the DHCP lease is released and a new IP may be assigned from the pool.

> **Interview trap:** "Does EC2 know its public IP?"
> No. The OS only knows its private IP. The public IP is a 1:1 NAT at the Internet Gateway. `curl ifconfig.me` works because the response comes back through the IGW, but `ip addr` will never show the public IP.

## AWS Internal Networking (VPC Routing)

Every VPC has an implicit router at the first IP of the CIDR (e.g., `10.0.0.1`). The route table controls where packets go:

```
Destination       Target
0.0.0.0/0    →   igw-xxxxxx       (internet traffic)
10.0.0.0/16  →   local            (VPC-internal traffic)
```

For private subnets:
```
Destination       Target
0.0.0.0/0    →   nat-xxxxxx       (outbound via NAT Gateway)
10.0.0.0/16  →   local            (VPC-internal traffic)
```

---

# 6. Why Same IP Cannot Exist on Two Running Instances

## ARP and IP Uniqueness

ARP (Address Resolution Protocol) maps an IP address to a MAC address. Every device on the network learns: "IP X belongs to MAC Y."

If two instances had the same IP:
1. Device A sends ARP request: "Who has 10.0.1.10?"
2. Both Instance 1 and Instance 2 reply with their MACs
3. The network receives conflicting ARP responses
4. Packet delivery becomes unpredictable — random one of the two instances gets each packet
5. Both instances are effectively broken

## VPC Routing Prevents This

AWS's VPC router maintains an internal table mapping each private IP to exactly one ENI:

```
10.0.1.10  →  eni-0a1b2c3d  (Instance A)
10.0.1.11  →  eni-0e5f6g7h  (Instance B)
```

When you try to assign `10.0.1.10` to a second instance while the first still owns it:
- AWS API call returns an error: `InvalidIPAddress.InUse`
- The assignment is blocked before it reaches the hypervisor
- No conflicting ARP ever occurs

## Packet Routing Diagram

```
Internet User
      │
      ▼
Internet Gateway
      │
      │  Destination: 13.235.100.50 (Elastic IP)
      │  IGW NAT: 13.235.100.50 → 10.0.1.10
      │
      ▼
VPC Router
      │
      │  Route table: 10.0.1.0/24 → Public Subnet
      │  ARP table:   10.0.1.10   → eni-0a1b2c3d (MAC: 02:42:ac:11:00:02)
      │
      ▼
ENI: eni-0a1b2c3d → EC2 Instance A

   [AWS blocks any attempt to assign 10.0.1.10 to a second ENI]
```

---

# 7. Ways to Replicate EC2

## Method 1: Launch from AMI

**What it does:** Creates a new instance with identical OS, software, and configuration.

**Steps:**
1. Create AMI from source instance (`create-image`)
2. Launch new instance from that AMI, specifying same subnet, SG, IAM role, instance type

**What is preserved:** OS, installed packages, files, EBS volumes, instance configuration
**What is NOT preserved:** Private IP (may differ), public IP (new), ENI (new), instance ID

**Best for:** Identical software stack, environment parity, ASG launch templates

---

## Method 2: Move ENI (Same Private IP + MAC)

**What it does:** Detaches the ENI from Instance A and attaches it to Instance B.

**Critical constraint:** Secondary ENIs only. The primary ENI (eth0) cannot be detached from a running instance.

**Steps:**
1. Stop both instances (or the source instance)
2. Detach ENI from Instance A
3. Attach ENI to Instance B
4. Start Instance B

**What is preserved:** Private IP, MAC address, Security Groups, secondary private IPs, Elastic IP associations

**Best for:** Software licensing tied to MAC, same private IP for internal service calls

---

## Method 3: Reuse Released Private IP

**What it does:** After Instance A is terminated (releasing its IP), you specify that exact IP when launching Instance B.

**Steps:**
1. Terminate (or stop) Instance A and confirm IP is released
2. Launch Instance B with `--private-ip-address 10.0.1.10`

**Caveat:** There is a small race window where another instance could grab the IP. For critical workflows, use ENI reuse instead.

**Best for:** DR scenarios where the original instance is fully gone and you want the same internal DNS/IP

---

## Method 4: Elastic IP (Same Public IP)

**What it does:** Moves a static public IP from one instance to another.

**Steps:**
1. Disassociate EIP from Instance A
2. Associate EIP with Instance B

**What is preserved:** Public IP (and thus external DNS/A records pointing to it)
**What is NOT preserved:** Private IP, MAC, ENI

**Best for:** External-facing services where users/DNS point to a known public IP, failover without DNS TTL wait

---

## Method 5: Launch Template

**What it does:** Encodes all instance configuration in a versioned template. New instances are always consistent.

**Steps:**
1. Create Launch Template capturing: AMI, instance type, subnet, SG, IAM role, user data, EBS config, tags
2. Launch instances from the template (manually or via ASG)

**Best for:** Repeatability, Auto Scaling Groups, blue/green deployments, standardized fleet

---

## Comparison Table

| Method | Same Private IP | Same Public IP | Same MAC | Same OS | Same SGs | Downtime Needed |
|---|---|---|---|---|---|---|
| AMI Launch | No (may differ) | No | No | Yes | Yes (specify) | No |
| Move ENI | Yes | Yes (if EIP) | Yes | No | Yes | Yes (secondary) |
| Reuse Released IP | Yes | No | No | No | Yes (specify) | Yes (old must be gone) |
| Elastic IP | No | Yes | No | No | Yes (specify) | No (brief disassoc) |
| Launch Template | No | No | No | Yes | Yes | No |

---

# 8. Production Decision Matrix

```
What do you need to preserve?
│
├── Same OS + software stack?
│       └── Create AMI → Launch from AMI
│
├── Same public IP (external users, DNS)?
│       └── Use Elastic IP → Disassociate → Associate
│
├── Same private IP (internal service calls, license)?
│       ├── Old instance still exists?
│       │       └── Move secondary ENI
│       └── Old instance is terminated?
│               └── Specify --private-ip-address at launch
│
├── Same MAC address (software licensing)?
│       └── Move ENI (MAC travels with ENI)
│
├── Repeatable / fleet-scale launches?
│       └── Launch Template + Auto Scaling Group
│
└── High Availability (live traffic)?
        └── ALB + ASG + Multi-AZ + AMI-based instances
            (No single instance matters; the fleet is the service)
```

---

# 9. End-to-End Production Traffic Architecture

> **Why this chapter exists:** The replication chapters above address "how do I clone an instance?" This chapter answers the more important question: **if your EC2 is delivering real traffic, what should the architecture look like?**
>
> A lone EC2 instance serving traffic directly is a production anti-pattern. This chapter shows the correct production model.

## The Problem with a Naked EC2

```
Internet → EC2  (WRONG for production)
```

Problems:
- Single point of failure (instance dies → service down)
- Auto-assigned public IP changes on restart
- No TLS termination layer (or you manage certs on the OS)
- No DDoS protection
- No caching / CDN
- Scaling requires DNS changes
- Every deployment requires downtime

## The Production Model

```
Internet
   │
   ▼
Route 53 (DNS + health checks)
   │
   ├──► CloudFront (CDN, edge caching, HTTPS)
   │         │
   └──► WAF (DDoS, OWASP rules)
             │
             ▼
     Application Load Balancer   ← lives in PUBLIC subnet
             │
     ┌───────┴────────┐
     │                │
     ▼                ▼
EC2 (AZ-a)      EC2 (AZ-b)     ← live in PRIVATE subnet
     │                │
     └───────┬────────┘
             │
    ┌────────┼──────────┐
    ▼        ▼          ▼
   RDS   ElastiCache   S3
  (Multi-AZ) (Redis)  (VPC endpoint)
             │
             ▼
   CloudWatch · CloudTrail · VPC Flow Logs · GuardDuty
```

## Layer-by-Layer Explanation

### Layer 1: Route 53 (DNS)

Route 53 is where every request starts. Users resolve `app.yourcompany.com` here.

**Key configurations:**
- **Alias record** pointing to ALB DNS name (not IP) — updates automatically if ALB IPs change
- **Health checks** on ALB or EC2 endpoints — Route 53 stops routing to unhealthy targets
- **Routing policies:** Simple, Weighted (A/B testing), Latency-based (multi-region), Failover (active/passive DR)

```
app.yourcompany.com  →  ALIAS  →  alb-1234.ap-south-1.elb.amazonaws.com
```

### Layer 2: CloudFront + WAF

**CloudFront (CDN):**
- Serves static assets (JS, CSS, images) from 400+ edge locations globally
- Caches API responses where appropriate (with cache-control headers)
- Terminates HTTPS at the edge — your ALB can use HTTP internally
- Reduces load on your EC2 instances dramatically

**WAF (Web Application Firewall):**
- Blocks OWASP Top 10 attacks: SQL injection, XSS, CSRF, path traversal
- Rate limiting per IP (prevents brute force)
- Geo-blocking (block entire countries if needed)
- Custom rules for your application

**AWS Shield:**
- Standard (free): L3/L4 DDoS protection on all AWS resources
- Advanced (paid): L7 protection, DDoS cost protection, 24/7 DRT team

### Layer 3: Application Load Balancer (ALB)

The ALB is the most critical component for EC2-based services.

**ALB lives in the public subnet.** Your EC2s live in the private subnet. This is the key security boundary.

**ALB capabilities:**
- Listener rules: route `/api/*` to API target group, `/static/*` to S3 origin
- TLS termination: single certificate managed by AWS ACM (no manual cert renewal)
- Health checks: TCP, HTTP, HTTPS checks on a specific path (e.g., `/health`)
- Sticky sessions: route the same user to the same EC2 for session-dependent apps
- Connection draining: waits for in-flight requests to complete before deregistering an instance

**Target Group:**
```
Target Group: tg-app
  Protocol: HTTP
  Port: 8080
  Health check: GET /health → expect HTTP 200
  
  Targets:
    - i-0a1b2c3d (EC2 AZ-a, 10.0.10.10)  → Healthy
    - i-0e5f6g7h (EC2 AZ-b, 10.0.11.10)  → Healthy
```

### Layer 4: EC2 Instances (Private Subnet)

Your replicated EC2 instances run here. Key points:

- **No public IP** — they are not directly reachable from the internet
- **Security Group (SG-app):** Inbound only from ALB's security group on app port
- **IAM Role (ec2-app-role):** Allows S3 read, SSM access, CloudWatch put metrics
- **NAT Gateway:** Provides outbound internet access (for package installs, external API calls)
- **SSM Session Manager:** Remote access without SSH or bastion host — no port 22 open

**This is where replication matters:**
All EC2 instances in the target group are launched from the same AMI via a Launch Template. When scaling, the ASG launches identical instances. When replacing, the new instance registers with the ALB target group before the old one deregisters. Zero downtime.

### Layer 5: Data Layer (Private Subnet)

**RDS (Multi-AZ):**
- Primary DB in AZ-a, standby in AZ-b
- Automatic failover in 60–120 seconds if primary fails
- Security Group (SG-db): inbound only from SG-app on DB port
- KMS-encrypted at rest

**ElastiCache (Redis):**
- Session storage: user sessions survive EC2 instance replacements (stateless EC2s)
- Caching: database query results, computed values
- Security Group (SG-cache): inbound only from SG-app on port 6379

**S3 (via VPC Gateway Endpoint):**
- Static assets, backups, log archives
- VPC endpoint: traffic never leaves AWS network (no NAT Gateway cost for S3)
- Bucket policy: allow only from VPC endpoint ID

### Layer 6: Observability

**CloudWatch:**
- EC2 metrics: CPU, memory (with CloudWatch Agent), disk, network
- ALB metrics: RequestCount, TargetResponseTime, HTTPCode_ELB_5XX_Count
- Alarms: trigger SNS → PagerDuty / Slack when CPU > 80% for 5 minutes
- Log groups: application logs from EC2 via CloudWatch Agent

**CloudTrail:**
- Records every API call in your account (who did what, when, from where)
- Essential for compliance and incident investigation
- Store in S3 with S3 Object Lock (tamper-proof)

**VPC Flow Logs:**
- Captures metadata of all network traffic through your VPC
- Sent to CloudWatch Logs or S3
- Essential for debugging: "why can't EC2 reach RDS?" → check flow logs for REJECT entries

**GuardDuty:**
- Threat intelligence: detects crypto mining, unusual API calls, port scanning
- Analyzes VPC Flow Logs, CloudTrail, DNS logs automatically
- Findings sent to Security Hub / EventBridge for automated remediation

## Network Security Model

```
SG-alb     Inbound: 443 from 0.0.0.0/0
           Outbound: 8080 to SG-app

SG-app     Inbound: 8080 from SG-alb only
           Outbound: 5432 to SG-db, 6379 to SG-cache, 443 to 0.0.0.0/0 (via NAT)

SG-db      Inbound: 5432 from SG-app only
           Outbound: none

SG-cache   Inbound: 6379 from SG-app only
           Outbound: none
```

**Rule:** Each layer can only talk to the layer directly adjacent to it. The internet cannot reach EC2 directly. EC2 cannot reach RDS except on the DB port.

## Request Flow: What Happens When a User Hits Your Service

```
1. User types app.yourcompany.com in browser

2. DNS resolution:
   Browser → Route 53 → returns ALB DNS name

3. TLS handshake:
   Browser ↔ CloudFront/ALB (certificate from ACM)
   EC2 sees plain HTTP internally

4. WAF inspection:
   WAF checks request against OWASP rules
   If blocked → 403 returned immediately
   If allowed → forward to ALB

5. ALB routing:
   ALB checks listener rules
   Routes to target group based on path
   Health check confirms EC2 is healthy
   Selects EC2 instance (round-robin or least-outstanding-requests)

6. EC2 processing:
   Application receives HTTP request
   Checks Redis cache for data
   If miss: queries RDS
   Fetches assets from S3 if needed
   Returns HTTP response to ALB

7. Response path:
   EC2 → ALB → CloudFront (cached for future requests) → User

8. Observability:
   CloudWatch receives ALB + EC2 metrics
   VPC Flow Logs records network metadata
   CloudTrail records any IAM/API calls made during processing
```

## Auto Scaling + Replication in Production

In production, you don't manually clone EC2s. The Auto Scaling Group does it:

```
Auto Scaling Group
  Launch Template: lt-app-v3 (AMI: ami-0a1b2c3d, instance type: t3.medium)
  Min: 2  Desired: 4  Max: 10
  Target Tracking Policy: ALBRequestCountPerTarget > 1000
  Health Check: ELB (uses ALB health check, not just EC2 status)
  Target Group: tg-app

When load increases:
  ASG launches new EC2 from Launch Template
  New EC2 runs user data script (configure app)
  New EC2 registers with ALB target group
  ALB health check passes → traffic flows to new instance

When an instance fails:
  ALB health check fails → instance deregistered (no traffic)
  ASG detects unhealthy instance → terminates it
  ASG launches replacement from same Launch Template
  New instance registers → traffic restored
```

This is the production answer to "EC2 replication" — not manual AMI cloning, but a fleet maintained by ASG from a golden AMI.

---

# 10. Architecture Diagrams

## Diagram 1: Current Environment (Single EC2)

```
Internet
    │
    ▼
Internet Gateway (igw-xxx)
    │
    ▼
Public Subnet (10.0.1.0/24)
    │
┌───┴──────────────────────┐
│          EC2             │
│  ┌────────────────────┐  │
│  │        ENI         │  │
│  │  Private: 10.0.1.10│  │
│  │  Public: 54.x.x.x  │  │
│  │  SG: sg-0a1b2c3d   │  │
│  └────────────────────┘  │
│  IAM: ec2-role           │
│  EBS: /dev/xvda 20GB gp3 │
└──────────────────────────┘
```

## Diagram 2: Production Traffic Architecture

```
Users (Internet)
    │
    ▼
Route 53 ──────────────── health check
    │
    ├──► CloudFront (CDN) ──► WAF
    │                            │
    └────────────────────────────┘
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │  Application Load      │
                    │  Balancer (Public SN)  │
                    └─────────┬──────────────┘
                              │
                 ┌────────────┴─────────────┐
                 │                          │
                 ▼                          ▼
         ┌──────────────┐          ┌──────────────┐
         │   EC2 (AZ-a) │          │   EC2 (AZ-b) │
         │  Private SN  │          │  Private SN  │
         │  10.0.10.10  │          │  10.0.11.10  │
         └──────┬───────┘          └──────┬───────┘
                │                         │
                └──────────┬──────────────┘
                           │
              ┌────────────┼──────────────┐
              ▼            ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │  RDS     │  │ Elasticache│ │    S3    │
        │ Multi-AZ │  │  Redis   │  │(VPC Endp)│
        └──────────┘  └──────────┘  └──────────┘
              │            │              │
              └────────────┴──────────────┘
                           │
              ┌────────────┴──────────────┐
              ▼                           ▼
        CloudWatch                   GuardDuty
        CloudTrail                VPC Flow Logs
```

## Diagram 3: ENI Architecture

```
EC2 Instance (i-0a1b2c3d)
│
├── eth0 → Primary ENI (eni-primary)
│         ├── Primary IP: 10.0.1.10
│         ├── Public IP: 54.123.45.67 (auto, lost on stop)
│         ├── Elastic IP: 13.235.100.50 (static, stays)
│         ├── SG: sg-app
│         └── MAC: 02:42:ac:11:00:02
│
└── eth1 → Secondary ENI (eni-secondary) [optional]
          ├── Primary IP: 10.0.1.20
          ├── SG: sg-mgmt
          └── MAC: 02:42:ac:11:00:03  ← can be moved to another instance
```

## Diagram 4: Elastic IP Failover Architecture

```
Before Failover:
    EIP: 13.235.100.50 → EC2-A (primary)

Trigger:
    EC2-A health check fails
    Automation (Lambda / runbook) fires

After Failover:
    1. aws ec2 disassociate-address --association-id <eid-xxx>
    2. aws ec2 associate-address --instance-id i-EC2-B --allocation-id <eipalloc-xxx>

    EIP: 13.235.100.50 → EC2-B (replica)
    DNS: no change needed (same IP)
    Users: unaffected after brief (~5s) reassociation
```

## Diagram 5: DR Architecture (Cross-AZ)

```
Region: ap-south-1
│
├── AZ-a
│     ├── VPC: 10.0.0.0/16
│     ├── Public Subnet: 10.0.1.0/24
│     │     └── ALB + NAT Gateway
│     └── Private Subnet: 10.0.10.0/24
│           └── EC2 (Primary) + RDS Primary
│
└── AZ-b
      ├── Public Subnet: 10.0.2.0/24
      │     └── ALB (cross-AZ enabled) + NAT Gateway
      └── Private Subnet: 10.0.11.0/24
            └── EC2 (Replica) + RDS Standby

Failover:
  AZ-a goes down →
  ALB stops sending to AZ-a targets →
  All traffic routes to AZ-b EC2 →
  RDS promotes standby to primary →
  Service continues, RTO ~2 minutes
```

---

# 11. Hands-on Lab

## Prerequisites

- AWS account with IAM permissions (EC2, VPC, IAM, S3)
- AWS CLI configured (`aws configure`)
- An existing EC2 instance running a service

---

## Step 1: Inspect Existing EC2

Go to EC2 Console → Instances → Select your instance → collect all values below.

```bash
# Collect all instance metadata via CLI
INSTANCE_ID="i-0a1b2c3d4e5f6789a"  # Replace with your instance ID

aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].{
    InstanceId:InstanceId,
    ImageId:ImageId,
    InstanceType:InstanceType,
    VpcId:VpcId,
    SubnetId:SubnetId,
    PrivateIpAddress:PrivateIpAddress,
    PublicIpAddress:PublicIpAddress,
    IamInstanceProfile:IamInstanceProfile.Arn,
    State:State.Name,
    AvailabilityZone:Placement.AvailabilityZone,
    Architecture:Architecture
  }' \
  --output table
```

### Parameters to Collect and WHY Each Matters

| Parameter | Where to Find | Why It Matters |
|---|---|---|
| Instance ID | Console header | Identifies source; needed for AMI creation |
| AMI ID | Details tab | Needed if you want to know base OS; your new AMI will be based on this |
| VPC ID | Networking tab | New instance must go in the same VPC for private IP access |
| Subnet ID | Networking tab | Same subnet = same AZ, same CIDR, same route table |
| Security Groups | Security tab | Must replicate to ensure firewall rules are identical |
| IAM Role | Security tab | Application uses this for AWS API calls; missing role = app breaks |
| ENI ID | Networking tab | Needed for ENI reuse method |
| EBS Volume ID | Storage tab | Not moved; new EBS created from snapshot inside AMI |
| Architecture (x86_64/arm64) | Details tab | New instance type MUST match; can't run arm64 AMI on x86_64 |
| AZ | Details tab | ENI reuse requires same AZ |
| Private IP | Networking tab | Note for reuse if instance is being terminated |
| Key Pair | Details tab | Needed to SSH (or use SSM instead) |

---

## Step 2: Create AMI

An AMI (Amazon Machine Image) is a snapshot of your instance's root volume plus configuration metadata (architecture, virtualization type, block device mappings).

### No-Reboot AMI (Default)

```bash
aws ec2 create-image \
  --instance-id $INSTANCE_ID \
  --name "myapp-replica-$(date +%Y%m%d-%H%M)" \
  --description "Production replica of $INSTANCE_ID" \
  --no-reboot \
  --tag-specifications 'ResourceType=image,Tags=[{Key=Name,Value=myapp-replica},{Key=Environment,Value=production},{Key=CreatedBy,Value=DevOps}]'
```

### Reboot AMI (Consistent)

```bash
aws ec2 create-image \
  --instance-id $INSTANCE_ID \
  --name "myapp-consistent-$(date +%Y%m%d-%H%M)" \
  --description "Consistent snapshot with reboot" \
  # No --no-reboot flag = instance will reboot
```

### No-Reboot vs Reboot — Explained

| Option | Behavior | Consistency Level | When to Use |
|---|---|---|---|
| `--no-reboot` | Instance stays running; snapshot taken live | Crash-consistent (like pulling power cord) | Production instances that can't afford downtime |
| Reboot (default) | Instance reboots first; snapshot taken after clean shutdown | Application-consistent | When data integrity is more important than uptime |

**Crash-consistent:** All data written to disk is captured. In-flight writes that hadn't flushed may be missing. Databases may be in an inconsistent state — but modern databases recover from this.

**Application-consistent:** OS has flushed all buffers. Databases have written all pending transactions. This is what `--no-reboot` does NOT give you.

**Production recommendation:** Use `--no-reboot` with database backup procedures (pg_dump, mysqldump) run before AMI creation, OR flush + freeze the database before snapshotting.

### Wait for AMI to Become Available

```bash
AMI_ID=$(aws ec2 create-image --instance-id $INSTANCE_ID --name "myapp-replica" --no-reboot --query 'ImageId' --output text)

aws ec2 wait image-available --image-ids $AMI_ID
echo "AMI $AMI_ID is ready"
```

---

## Step 3: Launch New EC2 from AMI

### Why Use the Same Subnet?

Same subnet = same AZ, same CIDR block, same route table, same NACL. Your new instance will have the same routing rules and the same network reachability as the original.

### Why Use the Same Security Group?

Security groups are stateful firewall rules. If SG-app allows inbound on port 8080 from SG-alb, your new instance needs to be in SG-app for the ALB to reach it. Assigning a different SG means the ALB health check will fail.

### Why Use the Same IAM Role?

Your application code calls AWS APIs (S3, SSM, SecretsManager, etc.). These calls authenticate using the instance's IAM role. Without the correct role, your application will get `AccessDenied` errors.

### Why Use the Same Architecture?

An AMI built on x86_64 cannot run on arm64 Graviton instances and vice versa. AWS will reject the launch request.

```bash
# Variables from Step 1
SUBNET_ID="subnet-0a1b2c3d"
SG_ID="sg-0a1b2c3d"
IAM_ROLE_ARN="arn:aws:iam::123456789012:instance-profile/ec2-app-role"
INSTANCE_TYPE="t3.medium"          # Must match architecture of AMI
KEY_NAME="my-keypair"              # Or use SSM and omit this

aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type $INSTANCE_TYPE \
  --subnet-id $SUBNET_ID \
  --security-group-ids $SG_ID \
  --iam-instance-profile Arn=$IAM_ROLE_ARN \
  --key-name $KEY_NAME \
  --tag-specifications \
    'ResourceType=instance,Tags=[{Key=Name,Value=myapp-replica},{Key=Environment,Value=production}]' \
  --query 'Instances[0].InstanceId' \
  --output text
```

---

## Step 4: Networking Deep Dive

### Auto-Assigned ENI

When you launch an instance, AWS automatically creates a primary ENI (eth0) in the specified subnet. You do not create the ENI separately.

```bash
NEW_INSTANCE_ID="i-0new1234"

# Inspect the auto-created ENI
aws ec2 describe-network-interfaces \
  --filters "Name=attachment.instance-id,Values=$NEW_INSTANCE_ID" \
  --query 'NetworkInterfaces[0].{
    ENI:NetworkInterfaceId,
    PrivateIP:PrivateIpAddress,
    PublicIP:Association.PublicIp,
    MAC:MacAddress,
    SGs:Groups[*].GroupId,
    Subnet:SubnetId,
    State:Status
  }' \
  --output table
```

### Assign Elastic IP to New Instance

```bash
# Allocate a new EIP (skip if reusing existing)
EIP_ALLOC=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)

# Associate with new instance
aws ec2 associate-address \
  --instance-id $NEW_INSTANCE_ID \
  --allocation-id $EIP_ALLOC
```

### Move Existing EIP from Old to New Instance

```bash
# Get current association ID
ASSOC_ID=$(aws ec2 describe-addresses \
  --filters "Name=instance-id,Values=$INSTANCE_ID" \
  --query 'Addresses[0].AssociationId' \
  --output text)

# Disassociate from old instance
aws ec2 disassociate-address --association-id $ASSOC_ID

# Get allocation ID of the EIP
EIP_ALLOC=$(aws ec2 describe-addresses \
  --filters "Name=instance-id,Values=$INSTANCE_ID" \
  --query 'Addresses[0].AllocationId' \
  --output text)

# Associate with new instance
aws ec2 associate-address \
  --instance-id $NEW_INSTANCE_ID \
  --allocation-id $EIP_ALLOC
```

### Attach Secondary ENI (for ENI reuse method)

```bash
ENI_ID="eni-0a1b2c3d"  # The secondary ENI from original instance

aws ec2 attach-network-interface \
  --network-interface-id $ENI_ID \
  --instance-id $NEW_INSTANCE_ID \
  --device-index 1

# Inside the OS of the new instance, bring up the interface:
# sudo ip link set eth1 up
# sudo dhclient eth1
```

### Route Table and NACL

```bash
# Check route table associated with subnet
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=$SUBNET_ID" \
  --query 'RouteTables[0].Routes'

# Check NACL associated with subnet
aws ec2 describe-network-acls \
  --filters "Name=association.subnet-id,Values=$SUBNET_ID" \
  --query 'NetworkAcls[0].Entries'
```

---

## Step 5: Testing

### SSH (Not Recommended — Use SSM)

```bash
# Only if port 22 is open in SG
ssh -i ~/.ssh/my-keypair.pem ec2-user@<PUBLIC_IP_OR_EIP>
```

### SSM Session Manager (Recommended)

```bash
# No port 22 needed. IAM role must have AmazonSSMManagedInstanceCore policy.
aws ssm start-session --target $NEW_INSTANCE_ID
```

### Connectivity Tests Inside Instance

```bash
# Test internet access (outbound via NAT Gateway for private subnet)
curl -s https://checkip.amazonaws.com

# Test RDS connectivity
nc -zv <RDS_ENDPOINT> 5432 && echo "RDS reachable"

# Test ElastiCache
nc -zv <ELASTICACHE_ENDPOINT> 6379 && echo "Redis reachable"

# Test S3 (via VPC endpoint)
aws s3 ls s3://your-bucket --region ap-south-1

# Test application
curl -s http://localhost:8080/health

# Test ALB health check endpoint
curl -s http://<ALB_DNS>/health

# Check logs
sudo tail -f /var/log/myapp/application.log
sudo journalctl -u myapp -f
```

### DNS Verification

```bash
# Confirm private DNS resolves correctly
dig ip-10-0-10-10.ap-south-1.compute.internal

# Confirm public DNS (if EIP attached)
dig ec2-13-235-100-50.ap-south-1.compute.amazonaws.com

# Confirm ALB DNS
dig myalb-1234567890.ap-south-1.elb.amazonaws.com

# Confirm custom domain
dig app.yourcompany.com
```

---

## Step 6: Verification Checklist

```
INSTANCE
[ ] Instance state: running
[ ] Correct AMI ID used
[ ] Correct instance type (matching architecture)
[ ] Correct AZ (if ENI reuse method)

NETWORKING
[ ] Private IP assigned (correct if specified)
[ ] Public IP / EIP attached (if required)
[ ] ENI visible (eth0, eth1 if secondary)
[ ] Security groups match original
[ ] Source/Destination check setting correct

CONNECTIVITY
[ ] SSH or SSM session works
[ ] Application port responds (curl localhost:8080/health)
[ ] RDS port reachable (nc -zv RDS_ENDPOINT 5432)
[ ] Redis port reachable (nc -zv CACHE_ENDPOINT 6379)
[ ] Internet reachable (curl checkip.amazonaws.com)
[ ] S3 accessible (aws s3 ls)

ALB INTEGRATION (if applicable)
[ ] Instance registered in target group
[ ] ALB health check status: Healthy
[ ] Traffic visible in application logs
[ ] ALB access logs show 2xx responses

IAM
[ ] IAM role attached
[ ] aws sts get-caller-identity returns correct role
[ ] Application can call required AWS services

OBSERVABILITY
[ ] CloudWatch Agent running (if using custom metrics)
[ ] Application logs appearing in CloudWatch Log Group
[ ] No REJECT entries in VPC Flow Logs for expected traffic
[ ] GuardDuty findings: none (or expected)
```

---

# 12. AWS CLI Reference

## describe-instances

```bash
# Basic instance info
aws ec2 describe-instances --instance-ids i-0a1b2c3d

# Filter by tag
aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=production" \
  --query 'Reservations[*].Instances[*].{ID:InstanceId,IP:PrivateIpAddress,State:State.Name}'
```

## describe-network-interfaces

```bash
# All ENIs for an instance
aws ec2 describe-network-interfaces \
  --filters "Name=attachment.instance-id,Values=i-0a1b2c3d"

# ENI by private IP
aws ec2 describe-network-interfaces \
  --filters "Name=addresses.private-ip-address,Values=10.0.1.10"
```

## create-image

```bash
# Create AMI without rebooting
aws ec2 create-image \
  --instance-id i-0a1b2c3d \
  --name "myapp-$(date +%Y%m%d)" \
  --no-reboot
```

## run-instances

```bash
# Full launch with all parameters
aws ec2 run-instances \
  --image-id ami-0a1b2c3d \
  --instance-type t3.medium \
  --subnet-id subnet-0a1b2c3d \
  --security-group-ids sg-0a1b2c3d \
  --iam-instance-profile Name=ec2-app-role \
  --private-ip-address 10.0.1.10 \    # Optional: specify exact IP
  --key-name my-keypair \
  --user-data file://userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=myapp-replica}]'
```

## associate-address (Elastic IP)

```bash
# Associate EIP with instance
aws ec2 associate-address \
  --instance-id i-0new1234 \
  --allocation-id eipalloc-0a1b2c3d

# Disassociate EIP
aws ec2 disassociate-address \
  --association-id eipassoc-0a1b2c3d
```

## attach-network-interface

```bash
# Attach secondary ENI to instance at device index 1 (eth1)
aws ec2 attach-network-interface \
  --network-interface-id eni-0a1b2c3d \
  --instance-id i-0new1234 \
  --device-index 1
```

## modify-instance-attribute

```bash
# Disable source/destination check (for NAT/VPN instances)
aws ec2 modify-instance-attribute \
  --instance-id i-0a1b2c3d \
  --source-dest-check '{"Value": false}'

# Change security groups
aws ec2 modify-instance-attribute \
  --instance-id i-0a1b2c3d \
  --groups sg-0a1b2c3d sg-0e5f6g7h
```

## describe-security-groups

```bash
# Describe SG rules
aws ec2 describe-security-groups --group-ids sg-0a1b2c3d \
  --query 'SecurityGroups[0].{
    Name:GroupName,
    Inbound:IpPermissions,
    Outbound:IpPermissionsEgress
  }'
```

## create-tags

```bash
# Tag instance and its volumes
aws ec2 create-tags \
  --resources i-0a1b2c3d vol-0a1b2c3d \
  --tags Key=Environment,Value=production Key=Owner,Value=devops-team
```

## ALB Target Group Registration

```bash
# Register instance with target group
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --targets Id=i-0new1234,Port=8080

# Check target health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:...
```

---

# 13. Terraform Version

## Directory Structure

```
ec2-replica/
├── main.tf
├── variables.tf
├── outputs.tf
├── userdata.sh
└── terraform.tfvars
```

## variables.tf

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-south-1"
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  description = "Public subnet CIDRs per AZ"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnet_cidrs" {
  description = "Private subnet CIDRs per AZ"
  type        = list(string)
  default     = ["10.0.10.0/24", "10.0.11.0/24"]
}

variable "availability_zones" {
  type    = list(string)
  default = ["ap-south-1a", "ap-south-1b"]
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.medium"
}

variable "source_ami_id" {
  description = "AMI ID to replicate"
  type        = string
}

variable "environment" {
  type    = string
  default = "production"
}

variable "app_name" {
  type    = string
  default = "myapp"
}
```

## main.tf

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# ─── VPC ───────────────────────────────────────────────────────────────────

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.app_name}-vpc"
    Environment = var.environment
  }
}

# ─── INTERNET GATEWAY ──────────────────────────────────────────────────────

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.app_name}-igw" }
}

# ─── SUBNETS ───────────────────────────────────────────────────────────────

resource "aws_subnet" "public" {
  count                   = length(var.public_subnet_cidrs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name        = "${var.app_name}-public-${count.index + 1}"
    Environment = var.environment
    Tier        = "public"
  }
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name        = "${var.app_name}-private-${count.index + 1}"
    Environment = var.environment
    Tier        = "private"
  }
}

# ─── NAT GATEWAY ───────────────────────────────────────────────────────────

resource "aws_eip" "nat" {
  count  = length(var.public_subnet_cidrs)
  domain = "vpc"
  tags   = { Name = "${var.app_name}-nat-eip-${count.index + 1}" }
}

resource "aws_nat_gateway" "main" {
  count         = length(var.public_subnet_cidrs)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = { Name = "${var.app_name}-nat-${count.index + 1}" }
  depends_on = [aws_internet_gateway.main]
}

# ─── ROUTE TABLES ──────────────────────────────────────────────────────────

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = { Name = "${var.app_name}-public-rt" }
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table" "private" {
  count  = length(var.private_subnet_cidrs)
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = { Name = "${var.app_name}-private-rt-${count.index + 1}" }
}

resource "aws_route_table_association" "private" {
  count          = length(aws_subnet.private)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# ─── SECURITY GROUPS ───────────────────────────────────────────────────────

resource "aws_security_group" "alb" {
  name        = "${var.app_name}-sg-alb"
  description = "ALB: allow HTTPS from internet"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.app_name}-sg-alb" }
}

resource "aws_security_group" "app" {
  name        = "${var.app_name}-sg-app"
  description = "EC2 app: allow from ALB only"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.app_name}-sg-app" }
}

# ─── IAM ROLE ──────────────────────────────────────────────────────────────

resource "aws_iam_role" "ec2_app" {
  name = "${var.app_name}-ec2-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })

  tags = { Name = "${var.app_name}-ec2-role" }
}

resource "aws_iam_role_policy_attachment" "ssm" {
  role       = aws_iam_role.ec2_app.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_role_policy_attachment" "cloudwatch" {
  role       = aws_iam_role.ec2_app.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}

resource "aws_iam_instance_profile" "ec2_app" {
  name = "${var.app_name}-ec2-profile"
  role = aws_iam_role.ec2_app.name
}

# ─── LAUNCH TEMPLATE ───────────────────────────────────────────────────────

resource "aws_launch_template" "app" {
  name_prefix   = "${var.app_name}-lt-"
  image_id      = var.source_ami_id
  instance_type = var.instance_type

  iam_instance_profile {
    arn = aws_iam_instance_profile.ec2_app.arn
  }

  network_interfaces {
    associate_public_ip_address = false
    security_groups             = [aws_security_group.app.id]
    delete_on_termination       = true
  }

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = 30
      volume_type           = "gp3"
      encrypted             = true
      delete_on_termination = true
    }
  }

  metadata_options {
    http_tokens                 = "required"  # IMDSv2 only
    http_put_response_hop_limit = 1
  }

  user_data = base64encode(file("userdata.sh"))

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name        = "${var.app_name}-instance"
      Environment = var.environment
    }
  }

  lifecycle {
    create_before_destroy = true
  }
}

# ─── ALB ───────────────────────────────────────────────────────────────────

resource "aws_lb" "main" {
  name               = "${var.app_name}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  enable_deletion_protection = true

  tags = { Name = "${var.app_name}-alb", Environment = var.environment }
}

resource "aws_lb_target_group" "app" {
  name     = "${var.app_name}-tg"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    path                = "/health"
    protocol            = "HTTP"
    matcher             = "200"
  }

  tags = { Name = "${var.app_name}-tg" }
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  # certificate_arn = "arn:aws:acm:..." # Add your ACM certificate ARN

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# ─── AUTO SCALING GROUP ────────────────────────────────────────────────────

resource "aws_autoscaling_group" "app" {
  name                = "${var.app_name}-asg"
  vpc_zone_identifier = aws_subnet.private[*].id
  target_group_arns   = [aws_lb_target_group.app.arn]
  health_check_type   = "ELB"

  min_size         = 2
  max_size         = 10
  desired_capacity = 2

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 50
    }
  }

  tag {
    key                 = "Name"
    value               = "${var.app_name}-asg-instance"
    propagate_at_launch = true
  }

  tag {
    key                 = "Environment"
    value               = var.environment
    propagate_at_launch = true
  }

  lifecycle {
    create_before_destroy = true
  }
}

# ─── VPC FLOW LOGS ─────────────────────────────────────────────────────────

resource "aws_cloudwatch_log_group" "flow_logs" {
  name              = "/aws/vpc/flowlogs/${var.app_name}"
  retention_in_days = 30
}

resource "aws_flow_log" "main" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flow_logs.arn
  log_destination = aws_cloudwatch_log_group.flow_logs.arn
}

resource "aws_iam_role" "flow_logs" {
  name = "${var.app_name}-flow-logs-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "vpc-flow-logs.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "flow_logs" {
  name   = "${var.app_name}-flow-logs-policy"
  role   = aws_iam_role.flow_logs.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents", "logs:DescribeLogGroups", "logs:DescribeLogStreams"]
      Resource = "*"
    }]
  })
}
```

## outputs.tf

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}

output "alb_dns_name" {
  description = "Point your Route 53 ALIAS record here"
  value       = aws_lb.main.dns_name
}

output "target_group_arn" {
  value = aws_lb_target_group.app.arn
}

output "launch_template_id" {
  value = aws_launch_template.app.id
}

output "asg_name" {
  value = aws_autoscaling_group.app.name
}

output "nat_gateway_eips" {
  description = "Elastic IPs of NAT Gateways (whitelist in external firewalls)"
  value       = aws_eip.nat[*].public_ip
}
```

---

# 14. Edge Cases

## ✓ Instance Already Owns ENI

**Situation:** You try to attach ENI but it's already attached to a running instance.

**Error:** `InvalidParameterValue: The ENI is currently attached.`

**Solution:** Detach from source instance first. For secondary ENIs, this can be done while running. For the primary ENI (eth0), you must stop the instance.

---

## ✓ Private IP Already Allocated

**Situation:** You specify `--private-ip-address` but the IP is in use by another instance.

**Error:** `InvalidIPAddress.InUse`

**Solution:** Find which ENI owns that IP and either terminate that instance or use a different IP. Use ENI reuse method instead of IP reuse if the original instance still exists.

---

## ✓ Different Availability Zone

**Situation:** You try to attach an ENI from AZ-a to an instance in AZ-b.

**Error:** `InvalidParameterValue: The ENI is in a different availability zone.`

**Solution:** ENIs are AZ-scoped. You cannot move them across AZs. For cross-AZ replication, use AMI + Launch Template method, and accept a different private IP in the new AZ.

---

## ✓ Different Architecture

**Situation:** Source instance is arm64 (Graviton), you try to launch the AMI on x86_64 instance type.

**Error:** `InvalidAMIID.NotApplicable`

**Solution:** AMI architecture must match instance family architecture. Graviton AMIs (arm64) require M6g, C6g, R6g, T4g, etc. Intel/AMD AMIs (x86_64) require M5, C5, R5, T3, etc.

---

## ✓ Different Instance Family

**Situation:** Moving from T3 (burstable) to M5 (general purpose).

**No error, but:** Enhanced Networking (ENA) must be enabled on the AMI for M5. T3 instances have ENA enabled by default.

**Solution:** Verify ENA is enabled: `aws ec2 describe-images --image-ids $AMI_ID --query 'Images[0].EnaSupport'` Should return `true`.

---

## ✓ Different Nitro Generation

**Situation:** Old instance is C4 (Xen), new instance is C5 (Nitro). Same OS AMI.

**Risk:** EBS volume may use paravirtual drivers that don't exist on Nitro. The instance may fail to boot.

**Solution:** Before migrating to Nitro, install NVMe drivers (`nvme-cli`), update `/etc/fstab` to use UUID-based mounts, and verify the AMI's virtualization type is `hvm`.

---

## ✓ Secondary ENI

**Situation:** Instance has eth0 (primary) and eth1 (secondary). You want to move only eth1 to a new instance.

**Solution:** Secondary ENIs can be detached from running instances (if you set `DeleteOnTermination=false`). Detach eth1, attach to new instance. The new instance gets eth1's IP and MAC on eth1.

```bash
# Detach secondary ENI
ATTACHMENT_ID=$(aws ec2 describe-network-interfaces \
  --filters "Name=attachment.instance-id,Values=$OLD_INSTANCE_ID" \
  --query 'NetworkInterfaces[?Attachment.DeviceIndex==`1`].Attachment.AttachmentId' \
  --output text)

aws ec2 detach-network-interface --attachment-id $ATTACHMENT_ID

# Reattach to new instance
aws ec2 attach-network-interface \
  --network-interface-id $ENI_ID \
  --instance-id $NEW_INSTANCE_ID \
  --device-index 1
```

---

## ✓ Elastic IP Attached to Source

**Situation:** Source EC2 has an EIP and you're launching a replica. You want the replica to get the EIP.

**Solution:** Don't terminate source until replica is ready. Then: disassociate EIP from source → associate with replica. Cutover time is ~5 seconds.

---

## ✓ Spot Instances

**Situation:** Source is a Spot Instance; you want to replicate it.

**Risk:** Spot instances can be interrupted by AWS. If it's interrupted mid-AMI creation, the AMI may be corrupted.

**Solution:** Create AMI from a persistent On-Demand snapshot, or from a Spot Instance with interruption notices disabled. For production, run AMI creation during low-risk hours and verify the AMI before terminating the source.

---

## ✓ Auto Scaling Group

**Situation:** Instance is part of an ASG and you want to replace it with a replica.

**Solution:** Don't manually clone ASG instances. Instead:
1. Update the Launch Template to a new AMI version
2. Use Instance Refresh on the ASG (rolling replacement)
3. ASG will launch new instances from the new template and terminate old ones

```bash
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name myapp-asg \
  --preferences '{"MinHealthyPercentage": 50, "InstanceWarmup": 60}'
```

---

## ✓ ENA Enabled

**Situation:** Replica instance doesn't get the same network performance as source.

**Cause:** Enhanced Networking (ENA) is not enabled on the AMI.

**Solution:**
```bash
# Check if ENA is enabled on AMI
aws ec2 describe-images --image-ids $AMI_ID \
  --query 'Images[0].EnaSupport'

# Enable ENA on a stopped instance before creating AMI
aws ec2 modify-instance-attribute \
  --instance-id $INSTANCE_ID \
  --ena-support '{"Value": true}'
```

---

## ✓ Source/Destination Check

**Situation:** Replica instance is meant to be a NAT or VPN appliance. Traffic isn't forwarding.

**Solution:** Disable source/destination check on the ENI.

```bash
aws ec2 modify-instance-attribute \
  --instance-id $NEW_INSTANCE_ID \
  --source-dest-check '{"Value": false}'
```

---

## ✓ IPv6

**Situation:** Source instance has an IPv6 address; you want the replica to be reachable on IPv6.

**Steps:**
1. Ensure VPC and subnet have IPv6 CIDR blocks assigned
2. Launch instance with `--ipv6-addresses` or enable auto-assign IPv6 on subnet
3. Update Security Groups to allow IPv6 traffic
4. Update Route Table to add `::/0 → igw-xxx`

---

## ✓ Cross-Region

**Situation:** You want to replicate the EC2 in a different AWS region.

**Steps:**
1. Create AMI in source region
2. Copy AMI to target region: `aws ec2 copy-image --source-region ap-south-1 --source-image-id $AMI_ID --region us-east-1 --name "myapp-useast1"`
3. Launch in target region from copied AMI
4. Note: Private IP, VPC, SG, IAM role all need to be recreated in target region

---

## ✓ Cross-Account

**Situation:** Share AMI from account A and launch in account B.

**Steps:**
1. Share AMI: `aws ec2 modify-image-attribute --image-id $AMI_ID --launch-permission '{"Add": [{"UserId": "123456789012"}]}'`
2. If EBS is encrypted: share KMS key with target account
3. In target account: `aws ec2 describe-images --executable-users self --owners 098765432109`
4. Launch from shared AMI in target account

---

## ✓ KMS Encrypted EBS

**Situation:** Source instance has KMS-encrypted EBS. You create AMI and try to launch in another account.

**Error:** `You do not have permission to use the KMS key.`

**Solution:** The AMI snapshot is encrypted with your KMS key. Target account needs key access. Either:
1. Add target account to KMS key policy
2. Or copy the AMI with re-encryption using a key accessible in the target account: `aws ec2 copy-image --kms-key-id <target-account-key>`

---

# 15. Assumptions

This guide assumes:

| Assumption | Why It Matters |
|---|---|
| Same AWS Region | ENI reuse and AMI launch are region-scoped |
| Same VPC | Private IP reachability requires same VPC |
| Same AWS Account | AMI and key pair access assumed; cross-account adds steps |
| IAM permissions available | EC2, VPC, IAM, ELB, ASG permissions required |
| AMI accessible | AMI must be in same region; shared if cross-account |
| Security Groups exist | SG IDs referenced at launch must pre-exist |
| Subnet has available IPs | `/24` subnet has 251 usable IPs; large fleets may exhaust |
| KMS key accessible | Encrypted EBS requires KMS key access |
| Instance type available | Some instance types are not available in all AZs |
| Service quotas not exceeded | Default limit: 32 vCPUs for On-Demand; request increase if needed |

---

# 16. Failure Scenarios

## ENI Detach Fails

**Cause:** Trying to detach primary ENI (eth0), or instance is in an invalid state.

**Error:** `OperationNotPermitted: You may not detach the primary network interface.`

**Recovery:** For the primary ENI, use the Elastic IP move method instead of ENI move. For secondary ENI, ensure the instance is not in a pending or shutting-down state.

---

## AMI Creation Fails

**Cause:** Insufficient EBS snapshot storage quota, S3 issues, or API error.

**Error:** `InternalError` or `ResourceLimitExceeded`

**Recovery:**
```bash
# Check AMI state
aws ec2 describe-images --image-ids $AMI_ID --query 'Images[0].State'

# If failed, delete the broken AMI and try again
aws ec2 deregister-image --image-id $AMI_ID
aws ec2 delete-snapshot --snapshot-id $SNAPSHOT_ID
```

---

## Subnet IP Exhausted

**Cause:** All 251 usable IPs in the subnet are allocated.

**Error:** `InsufficientFreeAddressesInSubnet`

**Recovery:**
1. Terminate unused instances to release IPs
2. Associate a secondary CIDR block with the VPC
3. Create a new subnet in the same AZ with a larger CIDR
4. Long-term: plan subnets with `/20` or larger for private workloads

---

## Security Group Deleted

**Cause:** SG was deleted between instance launch attempts.

**Error:** `InvalidGroup.NotFound`

**Recovery:** Recreate the SG with the same rules. If the original instance still exists, inspect its SGs to get the exact rules: `aws ec2 describe-security-groups --group-ids $SG_ID`

---

## KMS Encrypted EBS — Access Denied

**Cause:** IAM role or account doesn't have permission to use the KMS key.

**Error:** `AccessDeniedException: User is not authorized to use key`

**Recovery:**
```bash
# Check which KMS key is used
aws ec2 describe-volumes --volume-ids $VOL_ID \
  --query 'Volumes[0].KmsKeyId'

# Add IAM role to KMS key policy in KMS console
# Or use aws kms put-key-policy
```

---

## Launch Quota Exceeded

**Cause:** AWS account's vCPU On-Demand quota reached.

**Error:** `InstanceLimitExceeded` or `VcpuLimitExceeded`

**Recovery:**
```bash
# Check current limits
aws service-quotas get-service-quota \
  --service-code ec2 \
  --quota-code L-1216C47A  # Running On-Demand Standard instances

# Request increase
aws service-quotas request-service-quota-increase \
  --service-code ec2 \
  --quota-code L-1216C47A \
  --desired-value 64
```

---

## ALB Health Check Failing on New Instance

**Cause:** Application not started, wrong port, SG blocking ALB.

**Diagnosis:**
```bash
# Check target health reason
aws elbv2 describe-target-health \
  --target-group-arn $TG_ARN \
  --query 'TargetHealthDescriptions[*].{ID:Target.Id,State:TargetHealth.State,Reason:TargetHealth.Reason,Description:TargetHealth.Description}'

# Common reasons:
# Health checks failed → App not listening on port 8080
# Request timeout → SG-app not allowing ALB SG on port 8080
# Target.NotInUse → Instance not registered, wrong port
```

---

# 17. Best Practices

## Naming Conventions

```
VPC:      {app}-{env}-vpc                     myapp-prod-vpc
Subnet:   {app}-{env}-{tier}-{az}             myapp-prod-private-az-a
SG:       {app}-sg-{tier}                     myapp-sg-app
EC2:      {app}-{env}-{role}-{index}          myapp-prod-web-01
ENI:      (auto-named, tag it)                myapp-prod-eni-web-01
AMI:      {app}-{env}-{YYYYMMDD}-{HHMM}       myapp-prod-20250601-1430
LT:       {app}-lt-{version}                  myapp-lt-v3
ASG:      {app}-asg                           myapp-asg
ALB:      {app}-alb                           myapp-alb
TG:       {app}-tg-{service}                  myapp-tg-api
```

## Tagging Strategy

Always tag:

```
Name        = resource display name
Environment = production | staging | dev
Application = myapp
Owner       = devops-team
CostCenter  = backend-infra
CreatedBy   = terraform | manual | automation
CreatedOn   = 2025-06-01
```

Enforce via AWS Config rule or SCP (Service Control Policy).

## Security Best Practices

- **Never SSH directly** — use SSM Session Manager
- **Never assign public IPs to EC2 in production** — use ALB in public subnet, EC2 in private subnet
- **Enforce IMDSv2** on all instances (prevents SSRF attacks from stealing instance credentials)
- **Encrypt all EBS volumes** — enable by default at account level
- **Least-privilege IAM roles** — never attach AdministratorAccess to EC2
- **Rotate credentials** — use IAM roles, not access keys, on EC2

## Networking Best Practices

- Plan CIDR blocks for growth: use `/16` for VPC, `/20` or larger for private subnets
- One NAT Gateway per AZ (two NAT GWs for two AZs) — single NAT GW is a single point of failure
- Use VPC endpoints for S3 and DynamoDB to avoid NAT Gateway costs
- Enable VPC Flow Logs from day one — you'll need them for debugging
- Use Security Group references (SG-to-SG), not CIDR ranges, for inter-tier rules

## Cost Optimization

- **Savings Plans / Reserved Instances** for baseline EC2 capacity
- **Spot Instances** for fault-tolerant workloads (ASG mixed instances policy)
- **Delete unattached EIPs** — charged when not associated with a running instance
- **NAT Gateway costs** — use VPC endpoints for AWS services, PrivateLink for others
- **AMI cleanup** — deregister old AMIs and delete associated snapshots

## Automation

- All infrastructure via Terraform (or CDK) — no manual console changes in production
- AMI building via Packer or EC2 Image Builder — not manual
- Instance replacement via ASG Instance Refresh — not manual termination
- EIP failover via Lambda + EventBridge — not manual

---

# 18. Security Deep Dive

## Security Groups vs Network ACLs

| Property | Security Group (SG) | Network ACL (NACL) |
|---|---|---|
| Attached to | ENI (instance level) | Subnet |
| Statefulness | Stateful | Stateless |
| Rules | Allow only | Allow + Deny |
| Rule evaluation | All rules evaluated | Rules evaluated in order (lowest number first) |
| Return traffic | Automatically allowed | Must explicitly allow both directions |
| Default | Deny all inbound, allow all outbound | Allow all (default NACL) |

**Stateful (SG):** If you allow inbound on port 8080, the response traffic is automatically allowed outbound — you don't need an explicit outbound rule.

**Stateless (NACL):** If you allow inbound on port 8080, you must also add an outbound rule allowing the ephemeral ports (1024–65535) for the response to get back.

### When to Use NACLs

NACLs are for subnet-level rules. Use them to:
- Block specific IP addresses or CIDR ranges (e.g., known bad actors)
- Add an extra layer of defense (defense in depth)
- Isolate subnets during incident response

## IAM Roles for EC2

**Never** use IAM user access keys on EC2. Use IAM roles instead.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::myapp-prod-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters"
      ],
      "Resource": "arn:aws:ssm:ap-south-1:*:parameter/myapp/prod/*"
    }
  ]
}
```

The EC2 instance uses IMDS (Instance Metadata Service) to obtain temporary credentials:
```
http://169.254.169.254/latest/meta-data/iam/security-credentials/ec2-app-role
```

Enforce IMDSv2 to prevent SSRF:
```bash
aws ec2 modify-instance-metadata-options \
  --instance-id $INSTANCE_ID \
  --http-tokens required \
  --http-put-response-hop-limit 1
```

## SSM Session Manager (No SSH)

Replace SSH with SSM:

```bash
# Start session (no key pair, no port 22 needed)
aws ssm start-session --target i-0a1b2c3d

# Port forwarding (for local debugging)
aws ssm start-session \
  --target i-0a1b2c3d \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["8080"],"localPortNumber":["8080"]}'
```

Requirements:
- IAM role has `AmazonSSMManagedInstanceCore`
- SSM Agent running (pre-installed on Amazon Linux 2, Ubuntu 20.04+)
- SSM VPC endpoint OR instance has internet access for SSM endpoint

## VPC Flow Logs

Flow log record format:
```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status

Example REJECT entry:
2 123456789012 eni-0a1b2c3d 10.0.10.10 10.0.20.5 54321 5432 6 1 40 1620000000 1620000060 REJECT OK
```

This tells you: EC2 at `10.0.10.10` tried to reach `10.0.20.5:5432` (RDS) and was **REJECTED** — likely a missing SG rule.

---

# 19. Troubleshooting Guide

## Decision Tree

```
Instance not reachable?
│
├── Can you ping private IP from within VPC?
│     │
│     ├── No → Check Security Group inbound rules
│     │         Check Route Table
│     │         Check NACL
│     │         Check instance state (running?)
│     │
│     └── Yes → Can you reach the application port?
│               │
│               ├── No → Is the app running? (systemctl status myapp)
│               │         Is it listening on the right port? (ss -tlnp)
│               │         Check Security Group for that port
│               │
│               └── Yes → Is ALB health check passing?
│                         │
│                         ├── No → Check health check path (/health)
│                         │         Check health check SG rule
│                         │         Check response code (must be 200)
│                         │
│                         └── Yes → Check DNS / Route 53 routing
│
Cannot reach internet from EC2 (private subnet)?
│
├── Check NAT Gateway exists in public subnet
├── Check private subnet route table: 0.0.0.0/0 → nat-xxx
├── Check NAT Gateway has EIP attached
└── Check NAT Gateway status (available?)
│
EC2 cannot reach RDS?
│
├── Check SG-db inbound: allows SG-app on DB port?
├── Check SG-app outbound: allows DB port to SG-db?
├── Check RDS is in same VPC
├── Check VPC Flow Logs for REJECT on port 5432
└── Check RDS security group
│
High latency / poor network performance?
│
├── Check if ENA is enabled on AMI and instance
├── Check if instance is in Placement Group (if required)
├── Check CPU credits (T3 instances burst)
└── Check CloudWatch NetworkIn/NetworkOut metrics
```

## Common Errors and Solutions

| Error | Cause | Solution |
|---|---|---|
| `InvalidIPAddress.InUse` | IP already assigned | Terminate owner or use ENI reuse |
| `InsufficientFreeAddressesInSubnet` | Subnet exhausted | Delete unused instances or expand subnet |
| `InvalidAMIID.NotApplicable` | Architecture mismatch | Use matching instance family |
| `OperationNotPermitted` (ENI) | Detaching primary ENI | Use Elastic IP method instead |
| `UnauthorizedOperation` | Missing IAM permission | Add EC2/VPC permission to calling role |
| `InvalidGroup.NotFound` | SG deleted | Recreate SG |
| ALB: `Target.FailedHealthChecks` | App not responding | Check app is running on correct port |
| ALB: `Target.Timeout` | SG blocking ALB→EC2 | Add inbound rule: SG-alb on app port |
| SSM: `TargetNotConnected` | SSM Agent or role issue | Check IAM role + SSM VPC endpoint |

---

# 20. Interview Questions

## Fundamentals

**Q1:** What is an ENI and how does it relate to an EC2 instance's IP address?

**A:** An ENI (Elastic Network Interface) is a virtual NIC attached to an EC2 instance. The IP address belongs to the ENI, not the instance directly. When you assign an IP to an instance, AWS creates an ENI with that IP in the specified subnet and attaches it to the instance. The instance can have multiple ENIs; each has its own private IP, security groups, and MAC address.

---

**Q2:** Why does an EC2 instance's public IP change when you stop and start it?

**A:** The auto-assigned public IP is allocated from AWS's dynamic pool at launch and released back to the pool when the instance stops. When it restarts, a different IP from the pool is assigned. The public IP is never stored on the instance or ENI — it's a 1:1 NAT mapping at the Internet Gateway. Elastic IPs avoid this because you own the allocation; it persists in your account regardless of instance state.

---

**Q3:** Can two running EC2 instances have the same private IP? Why or why not?

**A:** No. AWS's VPC router maintains a mapping of IP → ENI. If you attempt to assign an IP already owned by an active ENI, the API returns `InvalidIPAddress.InUse`. At the network level, duplicate IPs would cause ARP conflicts — both instances would respond to ARP requests for that IP, making routing unpredictable.

---

**Q4:** What is the difference between a Security Group and a Network ACL?

**A:** Security Groups are stateful, instance-level firewalls attached to ENIs. They allow rules only (no explicit deny) and return traffic is automatically allowed. NACLs are stateless, subnet-level firewalls with numbered rules that support both allow and deny. Because they're stateless, you must allow both inbound and outbound traffic explicitly, including ephemeral ports for return traffic.

---

**Q5:** What happens to an ENI when an EC2 instance is terminated?

**A:** The primary ENI (eth0) is deleted by default on termination. Secondary ENIs have a `DeleteOnTermination` flag that you set at attachment time — if set to false, the ENI persists after termination. This is useful for preserving IP addresses and MAC addresses for reuse.

---

## Architecture

**Q6:** Your company's EC2 instance is serving production traffic. The instance needs to be replaced. How do you do it with zero downtime?

**A:** This requires the ALB + ASG pattern:
1. Verify the replacement instance is built from the same Launch Template (or updated AMI)
2. Launch the new instance and register it with the ALB target group
3. Wait for ALB health checks to pass (instance status: Healthy)
4. Connection draining on the old instance (ALB waits for in-flight requests to complete — default 300s)
5. Deregister the old instance from the target group
6. Terminate the old instance
With an ASG configured with health check type ELB and instance refresh, steps 2–6 are automated.

---

**Q7:** A user complains that every time your EC2 restarts, their DNS records need updating. How do you fix this?

**A:** Two options:
1. Assign an Elastic IP — static public IP that survives restart. Point DNS A record to EIP.
2. Put an ALB in front — use Route 53 ALIAS record pointing to ALB DNS name. ALB DNS never changes even if underlying IPs rotate. This is the production-correct approach.

---

**Q8:** You need to replicate an EC2 instance including its private IP for a DR test. The original instance will be terminated. What's your approach?

**A:**
1. Note the private IP of the original instance
2. Create an AMI from the original instance
3. Terminate the original instance and confirm the IP is released
4. Launch new instance from the AMI, specifying `--private-ip-address` for the exact IP
5. Verify connectivity and application function

If the original instance cannot be terminated, use secondary ENI move instead.

---

**Q9:** Why should EC2 instances in production sit in a private subnet?

**A:** Private subnets have no direct route to the internet — there's no Internet Gateway in the route table. This means the instance cannot be reached directly from the internet, even if someone knows the private IP. All inbound traffic goes through the ALB (in the public subnet), which enforces TLS, WAF rules, and listener rule filtering. Outbound traffic from EC2 goes through a NAT Gateway — the EC2's IP is never exposed externally. This dramatically reduces attack surface.

---

**Q10:** Explain the traffic path from a user request to your EC2 application in full detail.

**A:** (Refer to Chapter 9 — Request Flow section for the full 8-step answer)

---

## Scenario-Based

**Q11:** Your application is running on a single EC2 in `ap-south-1a`. The AZ experiences an outage. What was wrong with your architecture and how do you fix it?

**A:** The architecture had a single AZ dependency — single point of failure. Fix:
1. Multi-AZ deployment: identical EC2 instances in both AZ-a and AZ-b
2. ALB spanning both AZs with cross-zone load balancing enabled
3. RDS Multi-AZ for the database
4. One NAT Gateway per AZ (not shared)
5. ASG with `vpc_zone_identifier` listing both private subnets
Now if AZ-a fails, the ALB stops routing to AZ-a targets and all traffic goes to AZ-b. RTO is near-zero.

---

**Q12:** You need to move a software license from an old EC2 to a new one. The license is tied to the MAC address. How?

**A:** The MAC address belongs to the ENI, not the instance. Solution: detach the secondary ENI from the old instance (or the primary ENI after stopping it) and attach it to the new instance. The new instance's eth0 or eth1 will now present the same MAC address. The license tool will see an identical hardware fingerprint. This works even if the instances are different types.

---

**Q13:** An EC2 instance in a private subnet cannot reach the internet for package updates. What do you check?

**A:**
1. Route table: does the private subnet have `0.0.0.0/0 → nat-xxx`?
2. NAT Gateway: is it in a public subnet? Is it in state `available`?
3. Does the NAT Gateway's public subnet have `0.0.0.0/0 → igw-xxx`?
4. Does the NAT Gateway have an Elastic IP?
5. SG outbound: does SG-app allow port 443 outbound?
6. VPC Flow Logs: check for REJECT on outbound port 443

---

**Q14:** You're asked to set up a forensic copy of a compromised EC2 instance without modifying it. How?

**A:**
1. Do NOT stop the instance — memory artifacts are lost on stop
2. Create an AMI with `--no-reboot` flag (live snapshot)
3. For memory forensics, consider using SSM Run Command to dump memory first
4. Launch the AMI as a forensic copy in an isolated VPC (no internet access, no production network)
5. Attach the original EBS snapshot (not volume) to a clean analysis instance
6. Original compromised instance remains untouched for live analysis

---

**Q15:** Your EC2 replica is in the ALB target group but health checks keep failing. Walk me through your diagnosis.

**A:**
1. `describe-target-health` → read the `Reason` field
2. "Health checks failed" → EC2 is reachable but `/health` returns non-200. Check application logs.
3. "Request timeout" → EC2 not responding. Check: is app running? Is it listening on the health check port?
4. "Connection refused" → Port closed. Check: SG-app inbound allows SG-alb on the health check port?
5. "Target.InvalidState" → Instance in wrong state. Check instance status in EC2 console.
6. VPC Flow Logs → look for REJECT from ALB SG IP to EC2 private IP on app port.

---

**Q16:** What is the difference between an AMI and a snapshot?

**A:** A snapshot is a point-in-time backup of an EBS volume. An AMI is a complete machine image that includes one or more snapshots (one per EBS volume attached) plus metadata: architecture, virtualization type, block device mappings, kernel ID. An AMI can be used to launch instances; a snapshot alone cannot. When you create an AMI, AWS creates snapshots of all attached EBS volumes automatically.

---

**Q17:** How does Auto Scaling ensure replicated EC2 instances are identical?

**A:** Through a Launch Template, which encodes: AMI ID, instance type, security groups, IAM role, user data, EBS configuration, and tags. Every instance the ASG launches is created from this template. When you update the AMI (e.g., after a patch or code deploy), you update the Launch Template version and trigger an Instance Refresh. The ASG replaces instances in a rolling fashion, ensuring at least N% remain healthy throughout.

---

**Q18:** Your company must ensure EC2 instances are always patched and that old, unpatched instances are replaced. How do you automate this?

**A:**
1. EC2 Image Builder: automated pipeline that builds a new AMI weekly, runs OS patches, installs agents, runs security scans (Inspector)
2. Launch Template: updated to new AMI after Image Builder completes
3. ASG Instance Refresh: triggered automatically via EventBridge rule when new AMI is tagged `patch-status: approved`
4. This gives you a continuously patched fleet with zero manual intervention

---

**Q19:** Explain the purpose of a NAT Gateway. Why do you need one per AZ?

**A:** A NAT Gateway provides outbound internet access for instances in private subnets without exposing them to inbound internet connections. It translates the private IP to its own Elastic IP for outbound traffic. You need one per AZ because: (a) AZ independence — if the AZ hosting the NAT Gateway fails, all other AZs lose internet access too; (b) inter-AZ data transfer costs — traffic crossing AZs to reach a single NAT Gateway incurs data transfer charges. One NAT Gateway per AZ avoids both problems.

---

**Q20:** What is IMDSv2 and why should you enforce it?

**A:** IMDS (Instance Metadata Service) at `169.254.169.254` provides EC2 metadata including temporary IAM credentials. IMDSv1 is a simple GET request — any code on the instance (including SSRF attack code) can call it. IMDSv2 requires a PUT request to first get a session token, then uses the token for metadata requests. SSRF attacks typically can't follow this two-step flow, preventing credential theft. Enforce via Launch Template: `HttpTokens=required`.

---

**Q21:** How would you design a zero-downtime deployment for an EC2-based application?

**A:** Blue/Green deployment:
1. Current fleet (Blue): EC2s running v1, registered in ALB target group
2. Build Green fleet: launch identical EC2s from new AMI (v2) via new Launch Template version
3. Register Green instances in a new target group
4. Create a new ALB listener rule routing a small % of traffic to Green (weighted target groups)
5. Validate: metrics, logs, error rates on Green
6. Shift traffic 100% to Green via ALB listener rule update
7. Deregister and terminate Blue fleet
Rollback: shift ALB rule back to Blue target group — takes seconds.

---

**Q22:** An EC2's private IP is `10.0.1.10`. You stop the instance for 30 minutes. What happens to the IP?

**A:** For most cases, the private IP is retained. AWS holds the IP reservation for a stopped instance as long as the instance exists (not terminated). The ENI still exists with the IP allocated. However, the auto-assigned public IP IS released on stop. So on restart: same private IP, new public IP (unless EIP is used).

---

**Q23:** What is the difference between stopping and terminating an EC2 instance from a networking perspective?

**A:**
- **Stop:** Instance shuts down, ENI persists, private IP is retained, EBS volumes persist, auto-public IP is released (EIP retained)
- **Terminate:** Instance is deleted, primary ENI is deleted (private IP released back to pool), EBS root volume deleted by default (additional volumes configurable), EIP disassociated (still in your account until you release it)

---

**Q24:** How does Route 53 health check interact with EC2 failover?

**A:** Route 53 can monitor endpoints (IP or domain) at configurable intervals. If a health check fails N consecutive times, Route 53 marks the record as unhealthy and stops routing DNS queries to it. With a Failover routing policy (Primary/Secondary records), Route 53 automatically starts returning the Secondary record when Primary is unhealthy. This gives DNS-level failover without any load balancer — useful for active/passive DR across regions.

---

**Q25:** When would you use an ENI with a secondary private IP instead of attaching a second ENI?

**A:** Use secondary private IPs when: you need multiple IPs on the same network interface for hosting multiple SSL certificates (each certificate can bind to a distinct IP), or for active/passive IP failover where you move one IP within the same subnet without moving the whole ENI. Use a second ENI when: you need to be on two different subnets simultaneously, or when the application needs separate network isolation for management vs. application traffic (eth0 for app, eth1 for mgmt on a separate SG).

---

**Q26:** What is the VPC flow log entry you'd look for to confirm that an EC2 instance's health check is being blocked?

**A:**
```
REJECT entry showing:
  srcaddr = ALB node IP (in public subnet range)
  dstaddr = EC2 private IP
  dstport = 8080 (or your health check port)
  action  = REJECT
```
This would indicate either SG-app is not allowing inbound from SG-alb, or a NACL on the private subnet is blocking the traffic.

---

**Q27:** Explain how a NAT Gateway knows to route return traffic back to the correct private instance.

**A:** The NAT Gateway maintains a connection translation table. When an EC2 instance at `10.0.10.10:54321` makes an outbound request, the NAT Gateway translates it to `EIP:randomPort` and records the mapping. When the response arrives at `EIP:randomPort`, the NAT Gateway looks up the translation table and forwards the packet back to `10.0.10.10:54321`. This is NAPT (Network Address and Port Translation).

---

**Q28:** You have a fleet of 20 EC2 instances behind an ALB. A new security patch must be applied. How do you patch without downtime?

**A:** ASG Instance Refresh with a patched AMI:
1. Run Systems Manager Patch Manager to build a new patched AMI, OR update the Launch Template with a new AMI built via EC2 Image Builder
2. `aws autoscaling start-instance-refresh --auto-scaling-group-name myapp-asg --preferences '{"MinHealthyPercentage": 70}'`
3. ASG replaces instances in batches (30% at a time), waiting for health checks to pass before proceeding
4. Complete rollout takes ~15 minutes for 20 instances with 60s warmup
5. Zero downtime because ALB keeps routing to healthy instances throughout

---

**Q29:** What is the maximum number of ENIs and private IPs an EC2 instance can have?

**A:** It depends on the instance type. Each instance type has documented limits for maximum ENIs and IPs per ENI. For example:
- `t3.micro`: 2 ENIs × 2 IPs each = 4 private IPs max
- `m5.xlarge`: 4 ENIs × 15 IPs each = 60 private IPs max
- `c5.18xlarge`: 15 ENIs × 50 IPs each = 750 private IPs max

Check: `aws ec2 describe-instance-types --instance-types m5.xlarge --query 'InstanceTypes[0].NetworkInfo'`

---

**Q30:** Compare Elastic IP with Auto-assigned Public IP. When would you use each?

**A:**
| | Elastic IP | Auto-assigned Public IP |
|---|---|---|
| Cost | Free when attached to running instance; charged when unattached | Free |
| Persistence | Survives stop/start | Released on stop |
| Predictability | Fixed, known in advance | Changes every restart |
| Movability | Can move between instances | Cannot |
| Use case | External services, DNS, licensing | Dev/test, non-critical workloads |

Production recommendation: don't use either directly. Use an ALB — it gives a stable DNS name that Route 53 can alias, and the underlying IPs are AWS-managed. Reserve EIPs for NAT Gateways and specific legacy integrations.

---

# 21. Summary and Cheat Sheet

## Key Concepts Summary

| Concept | One-Line Summary |
|---|---|
| VPC | Your private network inside AWS |
| Subnet | Subdivision of VPC, tied to one AZ |
| ENI | Virtual NIC; holds IP, SG, MAC |
| Private IP | Internal VPC address; persists across stop/start |
| Public IP | Dynamic AWS address; lost on stop |
| Elastic IP | Static public IP you own; survives stop/start |
| AMI | Machine image; OS + disk; used to launch clones |
| Security Group | Stateful, instance-level firewall |
| NACL | Stateless, subnet-level firewall |
| NAT Gateway | Outbound internet for private subnet; one per AZ |
| ALB | Layer 7 load balancer; routes by path/host/header |
| ASG | Fleet of identical EC2s; auto-replace on failure |
| Launch Template | Blueprint for consistent EC2 launches |
| Route 53 | DNS + health checks + failover routing |
| VPC Flow Logs | Network traffic metadata for debugging |

## Replication Method Quick Reference

```
Need same OS?        → AMI + run-instances
Need same public IP? → Move Elastic IP
Need same private IP → Old instance alive? → Move ENI
  (same VPC, AZ)      Old instance dead?  → --private-ip-address at launch
Need same MAC?       → Move ENI
Need consistency?    → Launch Template + ASG
```

## Production Architecture Summary

```
Internet → Route 53 → WAF/CloudFront → ALB (public subnet)
                                          ↓
                                 EC2 × N (private subnet, ASG)
                                          ↓
                               RDS (Multi-AZ) + Redis + S3
                                          ↓
                         CloudWatch + CloudTrail + VPC Flow Logs
```

## Security Checklist (Production)

```
[ ] EC2 in private subnet (no public IP)
[ ] ALB in public subnet (TLS termination)
[ ] SG-app allows inbound only from SG-alb
[ ] SG-db allows inbound only from SG-app
[ ] IMDSv2 enforced on all instances
[ ] EBS encrypted (account-level default)
[ ] IAM role least-privilege (no AdministratorAccess)
[ ] No port 22 open; SSM Session Manager used
[ ] VPC Flow Logs enabled
[ ] CloudTrail enabled (all regions)
[ ] GuardDuty enabled
[ ] Config rules for compliance drift detection
```

---

# 22. Mastery Checklist

```
NETWORKING FUNDAMENTALS
✔ Can explain VPC, Subnet, CIDR, and reserved IPs
✔ Can explain ENI and every field in its configuration
✔ Can explain how DHCP assigns an IP during EC2 launch
✔ Can explain why duplicate private IPs are impossible
✔ Can explain the difference between SG and NACL

REPLICATION
✔ Can replicate an EC2 using AMI method
✔ Can move an Elastic IP from one instance to another
✔ Can move a secondary ENI to preserve private IP and MAC
✔ Can specify a released IP for a new instance launch
✔ Can create and use Launch Templates for repeatable launches

PRODUCTION ARCHITECTURE
✔ Can design a multi-AZ architecture with ALB + ASG
✔ Can explain why EC2s should be in private subnets
✔ Can explain the role of NAT Gateway (and why one per AZ)
✔ Can design SG rules following the SG-reference model
✔ Can explain the complete request flow from user to EC2
✔ Can configure an ALB target group with health checks
✔ Can explain when to use Route 53 vs ALB for failover

SECURITY
✔ Can enforce IMDSv2
✔ Can configure SSM Session Manager instead of SSH
✔ Can write least-privilege IAM role policy for EC2
✔ Can explain VPC Flow Log entry format and read REJECT entries
✔ Can configure WAF rules for OWASP Top 10

AUTOMATION
✔ Can use AWS CLI for all major EC2 and ENI operations
✔ Can write Terraform for VPC, SG, EC2, ALB, and ASG
✔ Can trigger ASG Instance Refresh for zero-downtime patching

TROUBLESHOOTING
✔ Can diagnose ALB health check failures
✔ Can debug private subnet internet access issues
✔ Can read VPC Flow Logs to identify blocked traffic
✔ Can diagnose EC2-to-RDS connectivity failures
✔ Can diagnose ENI attach/detach errors

INTERVIEW READY
✔ Can answer all 30 interview questions with real-world context
✔ Can design production failover architecture on whiteboard
✔ Ready for AWS Solutions Architect Associate / DevOps Engineer interviews
```

---

*Guide version: 2.0 | Last updated: June 2025*
*Covers: EC2, ENI, VPC, ALB, ASG, Route 53, WAF, CloudFront, RDS, ElastiCache, S3, CloudWatch, CloudTrail, VPC Flow Logs, GuardDuty, Terraform, AWS CLI*
