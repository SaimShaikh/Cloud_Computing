# AWS Service Quotas Reference Guide
### VPC | Subnet | Peering | NACL | SG | EC2 (ENI/EBS/SG) | EFS | SQS | SNS | S3 | S3 Event Notifications

> Last verified: August 2026. Values marked **(hard cap)** cannot be increased via Service Quotas/Support regardless of request. All other values are soft quotas — file a Service Quota increase request to raise them.

---

## 1. VPC & Subnets

| Resource | Min | Default | Max-to-Max (Ceiling) | Scope |
|---|---|---|---|---|
| VPCs | 0 | 5 | Hundreds (no fixed hard cap; scales IGW quota with it) | Per Region |
| Subnets per VPC | 1 | 200 | 200 **(hard cap)** | Per VPC |
| IPv4 CIDR blocks per VPC | 1 | 5 | 50 | Per VPC |
| IPv6 CIDR blocks per VPC | 0 | 5 | 50 | Per VPC |
| Internet Gateways | 0 | 5 | Scales automatically with VPC quota | Per Region |
| Elastic IPs | 0 | 5 | 5,000 | Per Region |
| NAT Gateways | 0 | 5 | 5 per AZ **(hard cap)** | Per AZ |
| Private IPs per NAT Gateway | 1 | 8 | 16 | Per NAT Gateway |

---

## 2. VPC Peering

| Resource | Min | Default | Max-to-Max | Scope |
|---|---|---|---|---|
| Active peering connections | 0 | 50 | **125** | Per VPC |
| Pending/outstanding peering requests | 0 | 25 | Contact Support (no published ceiling) | Per account |
| Peered NAU (Network Address Usage) units — same Region | 0 | 128,000 | **512,000** | Per VPC |

> Note: Cross-Region peered VPCs do **not** count toward the NAU quota — only same-Region peers do.

---

## 3. Network ACL (NACL)

| Resource | Min | Default | Max-to-Max | Scope |
|---|---|---|---|---|
| NACLs | 0 | 200 | 200 **(hard cap)** | Per VPC |
| Rules per NACL (in + out combined) | 1 | 20 | **40** | Per NACL |
| NACLs a subnet can associate with | 1 | 1 | 1 **(hard cap — one NACL per subnet)** | Per subnet |

---

## 4. Security Groups (SG)

| Resource | Min | Default | Max-to-Max | Scope |
|---|---|---|---|---|
| Security groups | 0 | 2,500 | **5,000** | Per Region |
| **SGs attachable per ENI** | 1 | 5 | **16** | Per network interface |
| Inbound rules per SG | 0 | 60 | **200** | Per SG |
| Outbound rules per SG | 0 | 60 | **200** | Per SG |
| (Rules-per-SG × SGs-per-ENI) combined | — | 300 | **1,000 (hard cap)** | Per ENI |

> Rule/quota math: if you raise SGs-per-ENI to 16, your rules-per-SG quota is auto-capped so the product never exceeds 1,000.

---

## 5. EC2 — ENI / EBS / SG Attachment Limits (Per Instance)

| Resource | Min | Default | Max-to-Max | Notes |
|---|---|---|---|---|
| ENIs per instance | 2 | Varies by instance type (typically 2–15) | **Up to 15+** on largest types (e.g., m5.24xlarge = 15) | **Fixed by instance type — NOT adjustable** |
| EBS volumes per instance | 1 | ~28 (most Nitro instances) | **128** (M7i, M7a, C7i, M8-family and newer) | **Fixed by instance type — NOT adjustable** |
| SGs per instance (via primary ENI) | 1 | 5 | **16** | Same quota as SG-per-ENI (Section 4) |
| ENIs per Region | 0 | 5,000 | **100,000+** on request | Per Region |

> ENI and EBS-per-instance limits are hypervisor/instance-type constraints, not account-level Service Quotas. There is no increase request path — the only way to raise them is to choose a larger or newer-generation instance type.

**Quick reference — ENIs by instance size (examples):**

| Instance Type | Max ENIs |
|---|---|
| t3.micro / t3.small | 2 |
| m5.large | 3 |
| m5.xlarge | 4 |
| m5.4xlarge | 8 |
| m5.24xlarge | 15 |

---

## 6. EFS (Elastic File System)

| Resource | Min | Default | Max-to-Max | Scope |
|---|---|---|---|---|
| File systems | 0 | 1,000 | **10,000** | Per Region |
| Mount targets per VPC | 1 per AZ | = number of AZs used | Structurally capped at 1 mount target per AZ per file system (~6 AZs typical max per Region) | Per VPC |
| Access points per file system | 0 | 1,000 | **10,000** | Per file system |
| Elastic throughput (read) | — | — | 10 GiB/s | Per file system |

---

## 7. SQS (Simple Queue Service)

| Resource | Min | Default | Max-to-Max | Scope |
|---|---|---|---|---|
| Message size | 1 byte | 256 KB | 256 KB **(hard cap)** → 2 GB via S3 Extended Client Library | Per message |
| In-flight messages (Standard queue) | 0 | 120,000 | 120,000 **(hard cap)** | Per queue |
| In-flight messages (FIFO queue) | 0 | 20,000 | 20,000 **(hard cap)** | Per queue |
| Message retention period | 60 sec | 4 days | 14 days **(hard cap)** | Per queue |
| Visibility timeout | 0 sec | 30 sec | 12 hours **(hard cap)** | Per message |
| Batch size (SendMessageBatch) | 1 | 10 messages | 10 messages / 256 KB total **(hard cap)** | Per API call |

---

## 8. SNS (Simple Notification Service)

| Resource | Min | Default | Max-to-Max | Scope |
|---|---|---|---|---|
| Message size | 1 byte | 256 KB | 256 KB **(hard cap)** | Per message |
| Topics | 0 | 100,000 | Adjustable further on request | Per account/Region |
| Subscriptions per topic | 0 | 12,500,000 | Adjustable further on request | Per topic |
| SMS publish TPS | — | Varies by Region | Adjustable on request | Per Region |

---

## 9. S3 (Simple Storage Service)

| Resource | Min | Default | Max-to-Max | Scope |
|---|---|---|---|---|
| General purpose buckets | 0 | 10,000 | **1,000,000** | Per account |
| Object size | 0 bytes | — | 5 TB **(hard cap)** | Per object |
| Multipart upload parts | 1 | — | 10,000 **(hard cap)** | Per upload |
| PUT/COPY/POST/DELETE requests | — | 3,500 RPS per prefix | Scales automatically, unbounded with more prefixes | Per prefix |
| GET/HEAD requests | — | 5,500 RPS per prefix | Scales automatically, unbounded with more prefixes | Per prefix |

---

## 10. S3 Event Notifications

| Resource | Min | Default | Max-to-Max | Scope |
|---|---|---|---|---|
| Notification configuration | 1 | 1 XML config (holds many rules) | 1 **(fixed structure — not a raisable count)** | Per bucket |
| Filter rules per notification rule | 0 | 5 (prefix/suffix filters) | 5 **(hard cap)** | Per rule |
| Simultaneous native destination types | 1 | 3 (SNS, SQS, Lambda — one each) | Effectively unlimited fan-out **if routed through SNS topic or EventBridge** | Per bucket |

> Design pattern: for fan-out to more than one destination per event type, use the **S3 → SNS → multiple SQS subscribers** pattern rather than relying on native multi-destination config.

---

## Key Takeaways

- **Hard caps** (NACL rules structure, SQS/SNS message size, S3 object size, multipart parts, in-flight message counts) cannot be raised no matter how the Support case is framed — design around them, don't request increases for them.
- **ENI-per-instance and EBS-volumes-per-instance are NOT account quotas.** They're baked into the instance type/Nitro hypervisor. The only lever is choosing a different instance type/family.
- **SG-per-ENI × Rules-per-SG** is a linked quota pair capped at 1,000 combined — raising one lowers headroom on the other.
- EFS mount targets scale with **AZ count**, not a raisable number — plan multi-AZ subnet coverage accordingly.

---

## Checking Live Quota Values via CLI

```bash
# List all quotas for a service (e.g., VPC)
aws service-quotas list-service-quotas --service-code vpc

# Get a specific quota value (e.g., VPCs per Region)
aws service-quotas get-service-quota \
  --service-code vpc \
  --quota-code L-F678F1CE

# Request a quota increase
aws service-quotas request-service-quota-increase \
  --service-code vpc \
  --quota-code L-F678F1CE \
  --desired-value 10
```

---

## Troubleshooting: Common LimitExceeded Errors

| Error | Cause | Fix |
|---|---|---|
| `VpcLimitExceeded` | Hit default 5 VPCs/Region | Request VPC quota increase or delete unused VPCs |
| `AttachmentLimitExceeded` (EBS) | Instance type's EBS volume cap reached | Switch to instance type with higher volume limit (M7i/M8 family) |
| `NetworkInterfaceLimitExceeded` | Instance type's ENI cap reached | Switch to larger instance size/type |
| `SecurityGroupLimitExceeded` | 5 SGs already attached to ENI | Request SG-per-ENI increase (up to 16) or consolidate rules |
| `RulesPerSecurityGroupLimitExceeded` | 60 rules hit on one SG | Request increase (up to 200) or split rules across SGs |
| `MaxConfigLimitExceeded` (VPC Peering) | 50 active peerings hit | Request increase (up to 125) or consider Transit Gateway instead |
