
# AWS Transfer Family 


---

## 1. Scenario

Imagine your company works with external partners — banks, vendors, healthcare labs, auditors — who are used to sending and receiving files the "old way": via SFTP or FTP. They don't want to call an S3 API or install an AWS CLI. They just want to open FileZilla or WinSCP, type in a host, username, and password (or key), and drag-and-drop files like they've done for the last 20 years.

At the same time, your company wants those files to land directly in **S3** or **EFS**, so the rest of your modern AWS pipeline (Lambda triggers, analytics, backups) can pick them up immediately — without anyone maintaining a physical FTP server, patching an EC2 instance, or worrying about storage running out.

This is exactly the gap **AWS Transfer Family** fills: it gives you a fully managed, protocol-compliant file transfer front end, while the actual storage backend is S3 or EFS.

**Typical real-world scenario:**
A logistics company receives daily shipment manifest files from 50 different partner warehouses. Each partner uploads via SFTP using their own credents. Historically, this meant running and maintaining SFTP servers on EC2 (patching, scaling, monitoring, security hardening). With AWS Transfer Family, AWS manages the server; you just define users, permissions, and where each partner's files should land in S3 — organized automatically into per-partner folders.

---

## 2. What Is AWS Transfer Family?

AWS Transfer Family is a **fully managed AWS service that lets external users transfer files using standard protocols directly into or out of Amazon S3 or Amazon EFS**, without AWS having to manage any transfer server infrastructure themselves.

It supports four protocols, each exposed as a separate "server type":

| Protocol | Full Form | Common Use Case |
|---|---|---|
| **SFTP** | SSH File Transfer Protocol | Most common; secure, encrypted, key or password based |
| **FTPS** | FTP over SSL/TLS | Legacy systems needing encryption over FTP |
| **FTP** | File Transfer Protocol | Legacy, unencrypted — only within a VPC, never public |
| **AS2** | Applicability Statement 2 | B2B/EDI file exchange (common in healthcare, retail, finance) |

Under the hood, AWS Transfer Family is essentially a **managed control plane + data plane for protocol translation**: the client speaks SFTP/FTPS/FTP/AS2, but the data actually lives in S3 objects or EFS files.

---

## 3. Why Use It (The Core Value Proposition)

- **No server management** — no EC2 instances to patch, scale, or secure for the transfer layer itself.
- **Protocol compatibility without re-engineering partners** — external partners keep using the same SFTP/FTPS/AS2 clients they always have; you don't force them onto S3 APIs.
- **Storage flexibility** — files land natively in S3 (object storage, cheap, durable, integrates with the rest of AWS) or EFS (POSIX file system, good for legacy apps expecting a real filesystem).
- **Fine-grained access control** — each user/partner can be locked to their own "logical home directory," so Partner A can never see Partner B's files, even though everything might sit in the same bucket.
- **Built-in high availability & scaling** — AWS handles multi-AZ resilience and scaling the transfer endpoints automatically.
- **Auditing & integration** — transfer events can trigger Lambda (post-upload processing), log to CloudWatch/CloudTrail, and integrate with IAM for identity.
- **Custom hostnames & branding** — you can front it with your own domain name (e.g., `sftp.yourcompany.com`) instead of an AWS-generated endpoint.

---

## 4. When to Use AWS Transfer Family

Use it when:
- External partners/customers/vendors **require** SFTP/FTPS/FTP/AS2 specifically (contractually or technically) and cannot be moved to modern APIs.
- You are **decommissioning a self-managed SFTP/FTP server** (running on EC2 or on-prem) and want to remove operational overhead.
- You need **per-user isolation** into specific S3 prefixes or EFS paths (multi-tenant file drop scenarios).
- Files need to land in S3/EFS and immediately trigger downstream AWS processing (ETL, data lake ingestion, EDI processing).
- You need AS2 for B2B/EDI exchanges (e.g., with healthcare or retail trading partners) and don't want to build/manage AS2 software yourself.
- Compliance requires strong audit trails, encryption in transit, and centralized IAM-based access control for file transfers.

---

## 5. When NOT to Use It

Avoid it when:
- Your users/systems can speak to **S3 directly** (via SDK, CLI, or presigned URLs) — that will almost always be cheaper and simpler than standing up a protocol server.
- You need **very high-throughput, low-latency** file transfer at massive scale — consider AWS DataSync, S3 Transfer Acceleration, or Snow family for bulk/one-time large-scale migrations instead.
- You only need occasional, internal, one-off file sharing — a presigned S3 URL or a simple internal tool is far cheaper.
- Cost sensitivity is extreme and usage is very low-volume — Transfer Family has an hourly server charge that runs even with zero data transferred, so a handful of infrequent transfers may not justify it.
- You need real-time streaming data ingestion — this is a **file-transfer** service, not a streaming service (use Kinesis/MSK instead).
- Plain unencrypted FTP is being requested for internet-facing use — AWS actively restricts unencrypted FTP to VPC-internal endpoints only, for security reasons.

---

## 6. Advantages

- **Zero infrastructure management** for the transfer layer — no OS patching, no scaling groups to tune.
- **Native S3/EFS backend** — files are immediately usable by the rest of your AWS ecosystem.
- **Strong identity flexibility** — supports service-managed users, or integration with existing identity providers (Active Directory via AD Connector, LDAP, Okta, custom identity providers via API Gateway/Lambda).
- **Per-user logical isolation** — scoped IAM policies and "chroot-like" home directories mean tenants don't see each other's data.
- **Encryption in transit** built into every protocol variant offered publicly (SFTP/FTPS/AS2).
- **Event-driven automation** — S3 events + Lambda let you kick off processing the instant a file lands.
- **Custom hostname support** — professional branding for partner-facing endpoints (with your own TLS certificate via ACM).
- **Multi-protocol under one service** — if you need both SFTP and AS2, you don't need two totally separate systems.
- **Pay-as-you-go plus predictable hourly baseline** — no huge upfront investment compared to building your own HA SFTP cluster.

---

## 7. Disadvantages

- **Hourly cost regardless of usage** — every enabled server type incurs an hourly charge even if nobody transfers a single file that day.
- **Data transfer charges add up** — outbound/inbound data processed through Transfer Family is billed per GB, on top of normal S3/EFS storage costs.
- **Not ideal for very large-scale bulk migration** — designed for ongoing operational file transfer, not one-time massive dataset moves (DataSync/Snowball are better there).
- **Protocol limitations remain protocol limitations** — e.g., plain FTP is still insecure by nature and is deliberately restricted to VPC-only use by AWS.
- **Complexity in custom identity provider setups** — integrating a fully custom auth flow (via Lambda) adds architectural overhead versus just using service-managed users.
- **EFS backend has its own cost/complexity profile** — if you pick EFS instead of S3, you're now also managing EFS performance modes, throughput modes, and mount targets.
- **Learning curve for IAM scoping** — getting per-user "logical home directory" and least-privilege IAM policies exactly right takes care; misconfiguration can accidentally expose more than intended.

---

## 8. Billing Variables (What Actually Drives Cost)

AWS Transfer Family billing has **two main dimensions**:

1. **Server Endpoint Hourly Charge**
 - Charged **per protocol-enabled server**, per hour, regardless of usage.
 - If you enable SFTP, FTPS, and AS2 as three separate server endpoints, you are billed for **three separate hourly charges**.
 - This runs 24/7 unless you explicitly stop/delete the server.

2. **Data Transfer / Data Processed Charge**
 - Billed **per GB** of data uploaded and downloaded through the service.
 - This is *in addition to* whatever your S3 storage class or EFS storage/throughput costs are — Transfer Family billing sits on top of, not instead of, your underlying storage service's bill.

**Other cost-affecting factors to keep in mind:**
- **Underlying storage costs** — standard S3 storage pricing (or EFS storage + throughput pricing) applies separately, based on where files actually live.
- **Custom hostname / ACM certificate** — usually free via ACM, but Route 53 hosted zone costs may apply if you manage DNS there.
- **AS2 specific processing** — AS2 has its own considerations around MDN (Message Disposition Notification) processing that can factor into data processed costs.
- **Lambda/CloudWatch costs** — if you build automation off Transfer Family events (e.g., Lambda triggered on upload), those services bill separately per their own pricing models.
- **Idle servers still cost money** — a very common real-world cost trap is leaving unused server endpoints running; stopping a server (where supported) pauses the hourly charge.
- **Data egress across regions/internet** — standard AWS data transfer OUT pricing can still apply depending on where clients are connecting from and where data ultimately goes.

> Always check the official AWS Transfer Family pricing page for current numbers — hourly and per-GB rates vary by region and change periodically.

---

## 9. Architecture (Conceptual)

At a high level, AWS Transfer Family sits as a **managed protocol gateway** in front of your storage:

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/d1bd8738-f6b9-4e3e-8e6f-a4d3027df5a6" />

**Flow explained:**
1. A partner connects using their chosen protocol client to the Transfer Family endpoint (public internet-facing, VPC-internal, or via VPC endpoint).
2. Transfer Family authenticates the user — either through **service-managed users** (credentials stored directly in Transfer Family) or a **custom identity provider** (Active Directory, LDAP, Okta, or your own Lambda-backed authentication logic).
3. Once authenticated, Transfer Family assumes a **scoped IAM role** tied to that specific user, restricting them to their designated S3 prefix or EFS path (their "logical home directory").
4. The file is written directly into S3 or EFS — there is no intermediate "AWS server disk" where the file temporarily sits; it goes straight to the target storage.
5. Optionally, S3 event notifications fire on object creation, triggering Lambda functions, SNS topics, or SQS queues for downstream automated processing.

---

## 10. Core Components

| Component | Role |
|---|---|
| **Server** | The actual endpoint resource — one per protocol (SFTP/FTPS/FTP/AS2). This is the "thing" that incurs the hourly charge. |
| **Endpoint Type** | Determines reachability: **Public** (internet-facing), **VPC-hosted** (private, reachable within your VPC/via VPN/Direct Connect), or **VPC Endpoint** with internal DNS. |
| **Identity Provider** | How users are authenticated — **Service Managed** (Transfer Family stores users/keys directly) or **Custom Identity Provider** (backed by AD, LDAP, Okta, or your own Lambda function via API Gateway). |
| **User** | A defined identity allowed to connect — has a username, authentication method (SSH key / password, depending on protocol), and an assigned IAM role + home directory. |
| **IAM Role (per user/session)** | Scopes exactly what the connecting user can do in S3/EFS — least privilege is critical here. |
| **Home Directory / Logical Directory Mapping** | Defines which S3 prefix or EFS path a user lands in, and whether they see it as their "root" (chroot-style isolation) or with custom folder mappings. |
| **Managed Workflow** (optional) | A feature that lets you define a sequence of automated steps (e.g., copy, tag, decrypt, custom Lambda step) to run automatically after a file upload completes — without needing to write your own S3-event-triggered Lambda from scratch. |
| **Security Policy** | Controls which TLS/SSH ciphers, MACs, and key exchange algorithms are allowed on the server — important for compliance (e.g., disabling weak legacy ciphers). |
| **Custom Hostname** | A friendly, branded DNS name (via Route 53 + ACM certificate) instead of the default AWS-generated server endpoint. |
| **AS2 Profile / Partnership** (AS2-specific) | Defines the trading partner relationship, certificates, and MDN (Message Disposition Notification) settings for AS2 exchanges. |
| **CloudWatch Logs** | Captures connection attempts, authentication events, and file transfer activity for auditing/troubleshooting. |
| **CloudTrail** | Captures management-plane API calls (who created/modified servers, users, roles) for governance. |

---

## 11. Identity Provider Options (Deeper Note)

This is one of the most important design decisions in Transfer Family, so it deserves its own section:

- **Service Managed**: Simplest option. You create users directly in Transfer Family, each with an SSH public key (for SFTP) or a password (for FTPS/FTP). Good for smaller numbers of external partners.
- **AWS Directory Service (AD Connector / Microsoft AD)**: Lets you authenticate users against an existing Active Directory. Useful when partners/employees already exist in a corporate directory.
- **Custom Identity Provider (API Gateway + Lambda)**: The most flexible option — you write a Lambda function that authenticates against literally any backend (your own database, Okta, a third-party API) and returns the IAM role + home directory mapping dynamically. Adds complexity but is very powerful for large-scale, dynamic partner onboarding.

---

## 12. Key Concepts to Remember (Quick Mental Model)

- Transfer Family is a **protocol front door**, not a storage service itself — the data always physically lives in S3 or EFS.
- Cost = **(hourly server charge × number of protocol servers running) + (data processed per GB)** + underlying S3/EFS storage costs.
- Security boils down to two layers: **(1) protocol-level encryption** (SSH for SFTP, TLS for FTPS/AS2) and **(2) IAM-level scoping** (which prefix/path a given user's role can touch).
- Public FTP (unencrypted) is intentionally restricted by AWS to VPC-internal use only — it is never allowed as a public internet-facing endpoint, for obvious security reasons.
- Managed Workflows exist so you don't always have to hand-roll a Lambda-on-S3-event pattern for common post-upload tasks.
- Think of it as: **"I want my partners to keep using SFTP/FTPS/AS2 like they always have, but I never want to run or patch a Linux SFTP server again, and I want the files to land straight in my modern AWS pipeline."**

---
