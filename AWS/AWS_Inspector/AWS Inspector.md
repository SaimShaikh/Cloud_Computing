# AWS Inspector — Complete End-to-End Guide

> A single, consolidated reference covering everything discussed: what AWS Inspector is, what it scans (and what it doesn't), agent-based vs. agentless scanning, architecture, findings, real-world scenarios, production use cases, deployment, integrations, pricing, best practices, and a quick-reference summary.

---


## 1. What is AWS Inspector?

AWS Inspector is Amazon's fully managed **continuous vulnerability management** service.

It automatically discovers supported AWS workloads and continuously scans them for:

- Operating system vulnerabilities
- Common Vulnerabilities and Exposures (CVEs)
- Missing security patches
- Vulnerable software packages
- Vulnerable container images
- Vulnerable Lambda dependencies
- Network exposure (whether a vulnerable resource is reachable from the internet)

**Key idea:** Unlike a traditional scanner that you run once a week or month, Inspector rescans automatically whenever:

- A new CVE is published
- A workload changes (new package installed, new image pushed, code updated)
- A new supported resource is created

---

## 2. Why Use AWS Inspector?

- Continuous vulnerability monitoring (not a one-time scan)
- Automatic resource discovery — no manual onboarding of every server
- No manual scan scheduling
- Centralised findings across accounts and regions
- Risk-based prioritisation (not just a raw CVE list)
- Native integration with the rest of the AWS security stack
- Fits naturally into DevSecOps / CI-CD pipelines
- Improves compliance posture (PCI-DSS, HIPAA, ISO 27001, etc.)
- Reduces operational effort compared to running your own vulnerability scanner

---

## 3. Evolution: Inspector Classic vs Modern Inspector

### Inspector Classic (Legacy)
- Required the **AWS Inspector Agent** to be installed manually on every EC2 instance
- Manual "assessment runs" — you had to trigger scans yourself
- Used rule packages (CIS benchmarks, security best practices, network reachability, etc.)
- Limited automation — no continuous rescanning
- If you forgot to install the agent, the instance was simply not scanned

```
EC2 → Inspector Agent Installed → Assessment Run → Results Generated
```

### Modern AWS Inspector (Current)
- **No dedicated Inspector agent required**
- Continuous, always-on scanning
- Event-driven rescanning (triggers on new CVEs or resource changes)
- Uses **AWS Systems Manager (SSM)** to collect EC2 software inventory
- Natively supports:
  - Amazon EC2
  - Amazon ECR (container images)
  - AWS Lambda
  - Inspector Code Security (optional — scans source code before deployment)

```
EC2 → SSM Agent → Systems Manager Inventory → AWS Inspector → Vulnerability Analysis
```

> Most modern Amazon Linux AMIs (and many other AWS-supported images) already ship with the SSM Agent pre-installed.

---

## 4. AWS Inspector Architecture

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/512d6577-f43a-4166-81d4-04467e4e66f6" />


**Reading the diagram :**
1. Inspector continuously pulls in CVE / threat intel data from sources like the National Vulnerability Database (NVD) and AWS's own threat intelligence.
2. It automatically discovers EC2 instances, ECR images, and Lambda functions in your account.
3. It builds a software inventory for each, cross-checks it against known CVEs, and factors in network exposure (is this resource reachable from the internet?).
4. It generates findings with a risk score.
5. Findings flow to Security Hub (centralised view for the SOC team), EventBridge (for automated remediation workflows), and CloudWatch (metrics/alerting).
6. EventBridge can trigger Systems Manager Patch Manager to actually install the patch.
7. Once patched, Inspector automatically rescans and closes the finding.

---

## 5. What Infrastructure Inspector Scans

| AWS Service | Supported | What It Scans |
|---|---|---|
| ✅ Amazon EC2 | Yes | OS, installed packages, software vulnerabilities, network exposure |
| ✅ Amazon ECR | Yes | Container image vulnerabilities (OS packages + application dependencies) |
| ✅ AWS Lambda | Yes | Function dependencies, Lambda layers, vulnerable libraries |
| ✅ Source Code (Inspector Code Security) | Yes (optional) | Source code vulnerabilities and hardcoded secrets, before deployment |

### EC2 — Scans:
- Operating system
- Installed packages and libraries
- Package versions
- Known CVEs
- Network exposure (via Security Groups, Route Tables, IGW, ENIs, Public IPs)

### Amazon ECR — Scans container images for:
- Base operating system
- Image layers
- Python packages, Java libraries, Node.js modules, Go modules, Ruby gems, PHP packages, .NET dependencies

### AWS Lambda — Scans:
- Deployment package
- Lambda layers
- Runtime dependencies
- Third-party libraries

### Inspector Code Security (Optional) — Scans:
- Source code for vulnerabilities
- Hardcoded secrets
- Security issues before deployment (shift-left security)

---

## 6. What Infrastructure Inspector Does NOT Scan

Inspector does **not** directly scan:

| Service | Why Not | Use This Instead |
|---|---|---|
| Amazon S3 | Object storage, no OS | Macie, AWS Config |
| IAM | Identity & permissions, not software | IAM Access Analyzer |
| Amazon RDS | Managed database engine | Systems Manager (self-managed DBs), Security Hub, Config |
| DynamoDB | Fully managed NoSQL | AWS Config |
| VPC / Security Groups | Network configuration, not software | Network Access Analyzer, VPC Reachability Analyzer, AWS Config, Firewall Manager |
| Amazon EKS (cluster itself) | Cluster control plane not scanned | Inspector scans the container **images** in ECR that EKS runs |
| Amazon ECS (cluster itself) | Cluster itself not scanned | Inspector scans the **ECR images** used by ECS tasks |
| Aurora, EFS, FSx, CloudFront, API Gateway, Route 53, WAF, Shield | No OS/package layer to scan | Respective AWS security/config services |

> **Important nuance:** Inspector *does* use networking data (Security Groups, Route Tables, Internet Gateway, ENIs, Public IPs) to determine whether a vulnerable EC2 instance is internet-reachable — this affects the severity of a finding — but it does not scan networking *configuration* for misconfigurations itself.

---

## 7. Why Only Compute Workloads? (The Core Logic)

AWS Inspector is built to find **software vulnerabilities (CVEs)**. Only workloads that actually **run code** and **install software packages** are suitable for this type of scanning.

- **EC2** → Runs an operating system and applications → ✅ Can have vulnerable software
- **ECR** → Stores container images with OS packages and libraries → ✅ Can have vulnerable software
- **Lambda** → Runs application code with dependencies → ✅ Can have vulnerable libraries
- **S3 / RDS / DynamoDB** → Don't expose an operating system for you to manage → ❌ Nothing for Inspector to scan

### Mental model: Layers of AWS Security

```
                   AWS Infrastructure
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   Compute Layer      Identity Layer    Storage Layer
        │                  │                  │
  EC2  ECR  Lambda        IAM               S3
        │                  │                  │
   AWS Inspector    IAM Access Analyzer      Macie
```

### Worked Production Example

Suppose your architecture has:
- ALB
- 2 EC2 web servers
- Amazon ECR (stores Docker images)
- Amazon EKS (runs the containers)
- Amazon RDS
- S3
- IAM

Inspector will:
- ✅ Scan the EC2 instances
- ✅ Scan the Docker images in ECR (which EKS pulls)
- ❌ Not scan the EKS control plane itself
- ❌ Not scan the RDS database engine
- ❌ Not scan S3 buckets
- ❌ Not scan IAM policies

---

## 8. Agent-Based vs Agentless Scanning

This is one of the most important concepts to understand in vulnerability management.

### What is an Agent?

An agent is a small software program installed on a server that collects information and sends it to a management service — think of it as **a security employee living inside the server**, continuously monitoring and reporting back.

```
+----------------------------+
|        EC2 Instance        |
|                             |
|  Ubuntu / Apache / Docker  |
|                             |
|   +--------------------+   |
|   |  Security Agent    |   |
|   +--------------------+   |
+----------------------------+
            │
            ▼
      Sends information
            │
            ▼
      AWS Security Service
```

### Agent-Based Scanning

You must install software (an agent) on **every** server you want to monitor. The agent runs continuously inside the OS and collects:

- Installed software
- Running processes
- OS version
- CPU / memory usage
- File changes
- Installed patches
- Security logs

```
+----------------------+
|      EC2 Server      |
|                       |
|  Ubuntu / Apache /    |
|      Docker           |
|                       |
|  Security Agent       |
+----------+-----------+
           │  Collects Data
           ▼
     AWS Inspector
```

**Advantages:**
- ✅ Very detailed information
- ✅ Continuous, real-time monitoring
- ✅ Can inspect inside the OS deeply
- ✅ Detects software changes immediately

**Disadvantages:**
- ❌ Must install the agent everywhere
- ❌ Consumes CPU and memory
- ❌ Requires maintenance and version updates
- ❌ Hard to manage across thousands of servers

**Example — scaling pain:**
With 500 EC2 instances, agent-based scanning means:
```
EC2 1   → Install Agent
EC2 2   → Install Agent
EC2 3   → Install Agent
...
EC2 500 → Install Agent
```
Every single server needs the software installed and kept up to date.

### Agentless Scanning

No dedicated scanning software is installed inside the server. Instead, the cloud platform gathers information using **existing cloud APIs and services**.

```
               AWS Inspector
                      │
        AWS APIs / Existing Services
                      │
                      ▼
              EC2 Instance
```

Modern Inspector's agentless architecture:

```
                AWS Inspector
                      │
                      ▼
           AWS Systems Manager
                      │
                      ▼
              EC2 Instance
        (No dedicated Inspector agent)
```

**Advantages:**
- ✅ No installation required
- ✅ No dedicated-agent maintenance
- ✅ Lower operational effort
- ✅ Easy to enable across many instances at once
- ✅ No extra software consuming server resources

**Disadvantages:**
- ❌ May offer less depth than a specialised endpoint security agent
- ❌ Depends on supported cloud services and correct IAM permissions
- ❌ Only works for supported AWS workloads

### Real-World Analogy

**Agent-Based** = A security guard posted inside every building:
```
Building → Security Guard → Reports everything happening inside
```
The guard is always physically present.

**Agentless** = CCTV cameras monitored from a central security room:
```
Building → Cameras → Central Security Room → Monitoring
```
No guard is physically stationed inside, but the building is still watched.

### Comparison Table

| Feature | Agent-Based | Agentless |
|---|---|---|
| Software installation required | ✅ Yes | ❌ No dedicated scanner agent |
| Uses CPU/RAM on server | ✅ Yes | Minimal (existing services only) |
| Easy to deploy | ❌ No | ✅ Yes |
| Maintenance | High | Low |
| Scalability | Moderate | Excellent |
| Requires updates | Yes | No dedicated agent updates |
| Works outside cloud | Often yes | Usually cloud-specific |
| Used by Inspector Classic | ✅ Yes | ❌ No |
| Used by Modern AWS Inspector | ❌ No dedicated Inspector agent | ✅ Yes (with SSM for EC2) |

### Important Nuance

Modern Inspector is often described as "agentless" because it no longer needs the old, dedicated **Inspector Agent**. However:

> EC2 package inventory collection still relies on the **SSM Agent** being installed and running on the operating system. So it's more accurate to say Inspector removed *its own* agent requirement by piggybacking on Systems Manager — it isn't truly "zero-agent" at the OS level for EC2.

- **Inspector Classic (legacy):** Required the dedicated AWS Inspector Agent on every EC2 instance.
- **Modern AWS Inspector:** No dedicated Inspector agent. Integrates with AWS Systems Manager (SSM) for EC2 inventory, and uses native AWS integrations to continuously assess EC2, ECR images, and Lambda functions.

---

## 9. How AWS Inspector Works (Step-by-Step)

1. Enable Inspector in the account/region.
2. Inspector automatically discovers supported resources (EC2, ECR, Lambda).
3. Systems Manager provides software inventory for EC2.
4. Inspector compares that inventory against CVE databases.
5. A risk score is calculated per finding (CVSS + context like internet exposure).
6. Findings are generated.
7. Findings are sent to Security Hub / EventBridge / CloudWatch.
8. The security team (or an automated workflow) patches the vulnerability.
9. Inspector automatically rescans the resource.
10. The finding closes once remediation is confirmed.

---

## 10. Components

- **Resource Discovery** — automatically finds EC2, ECR, Lambda resources
- **Software Inventory** — via SSM for EC2, image layer analysis for ECR, dependency analysis for Lambda
- **CVE Intelligence** — continuously updated from NVD and AWS threat intel
- **Risk Analysis Engine** — combines CVSS score with contextual risk (e.g., internet exposure)
- **Findings Engine** — generates and tracks findings over their lifecycle
- **Continuous Monitoring** — re-triggers scans on new CVEs or resource changes
- **Reporting** — dashboards and exportable reports
- **Integrations** — Security Hub, EventBridge, SNS, CloudWatch, Systems Manager

---

## 11. Findings & Risk Scoring

Each finding contains:

- Resource ID
- CVE ID
- Severity
- Installed version
- Fixed version
- Detection time
- Exploitability
- Internet exposure
- Remediation recommendation

**Severity levels:** Critical, High, Medium, Low, Informational

**Risk score is influenced by:**
- CVSS base score
- Whether the resource is exposed to the internet
- AWS threat intelligence (is this CVE being actively exploited in the wild?)
- Public exploit availability

---

## 12. Advantages

- Fully managed — no infrastructure to run yourself
- Automatic resource discovery
- Continuous (not periodic) monitoring
- Deep native integration with the AWS security ecosystem
- Supports multi-account setups via AWS Organizations
- Prioritised, risk-contextualised findings (not just a raw CVE dump)
- Minimal maintenance overhead
- Scales cleanly from a handful of instances to thousands

---

## 13. Limitations

- Only scans supported compute workloads (EC2, ECR, Lambda, + optional code scanning)
- Does not replace antivirus / EDR (endpoint detection & response)
- Does not perform penetration testing
- Does not scan S3, IAM, RDS, DynamoDB, or other non-compute services
- Cost increases as workload scale increases
- Depth of visibility can be lower than a dedicated, specialised endpoint agent in some cases

---

## 14. Real-World Scenarios

These scenarios illustrate how Inspector behaves in situations you're likely to actually encounter.

### Scenario A — "We just launched a new EC2 fleet, is it safe?"
You launch 20 new EC2 web servers running Ubuntu with the SSM Agent pre-installed (as most modern AMIs have). As soon as they're running, Inspector auto-discovers them, pulls software inventory via SSM, and starts continuous CVE matching — no manual scan needed. If one of them has an outdated OpenSSL version with a known CVE, a finding appears within hours, not weeks.

### Scenario B — "A new critical CVE just got published for a library we use"
Suppose a new critical CVE is published today for a widely used compression library. You don't have to re-run anything manually — Inspector automatically re-evaluates every EC2 instance, every ECR image, and every Lambda function that has that library installed, and raises new findings immediately. This is the "continuous" part of continuous vulnerability management.

### Scenario C — "Our CI/CD pipeline pushes a new Docker image to ECR"
A developer pushes a new image tag to ECR as part of a deployment pipeline. Inspector automatically scans the new image layers for OS-level and application-level (Python/Node/Java/etc.) vulnerabilities before — or as — it gets deployed to ECS/EKS. If a critical vulnerability is found, this can be wired (via EventBridge) to block the deployment or alert the team.

### Scenario D — "We have a Lambda function with an old dependency"
Your Lambda function bundles an old version of a JSON-parsing library that has a known vulnerability. Inspector scans the function's deployment package and Lambda layers, flags the vulnerable dependency, and recommends the fixed version — without you needing to invoke or test the function.

### Scenario E — "Is this vulnerable server actually a real risk?"
Two EC2 instances both have the same CVE. One sits in a private subnet with no internet route; the other has a public IP and an open Security Group on port 443. Inspector's risk scoring will treat the internet-facing instance as significantly higher priority — this is the "network reachability" context in action, helping your team triage what to patch first instead of treating every CVE the same.

### Scenario F — "Why doesn't Inspector flag our misconfigured S3 bucket?"
Your S3 bucket is accidentally public. Inspector will **not** flag this — it isn't looking at S3 at all, because S3 has no OS/software layer. This is exactly the kind of finding Macie (sensitive data) or AWS Config (misconfiguration rules) is meant to catch instead. Understanding this boundary prevents a false sense of complete security coverage.

### Scenario G — "We use EKS — are our clusters protected?"
Your team runs workloads on EKS. Inspector does not scan the EKS control plane or worker node OS as an "EKS resource" directly — but since your containers are built from images stored in ECR, Inspector scans those **images**. So indirectly, your EKS-run containers are protected through the ECR scanning path, not through an "EKS scan" itself.

---

## 15. Production Use Cases

- **Banking** — continuous compliance evidence for regulators, prioritised patching of internet-facing systems
- **Healthcare** — HIPAA-aligned vulnerability management for EC2-hosted patient systems
- **Government** — compliance and audit trail via Security Hub + CloudTrail
- **SaaS companies** — protecting multi-tenant EC2/container fleets at scale
- **DevSecOps CI/CD** — blocking vulnerable container images before they reach production
- **E-commerce** — protecting containerised checkout/payment services running on ECS/EKS

**Typical automated pipeline:**
```
Developer → Push to ECR → Inspector Scan → Security Hub → EventBridge → Systems Manager → Patch → Rescan
```

---

## 16. End-to-End Workflow

```
Launch workload
      ↓
Automatic discovery
      ↓
Software inventory
      ↓
CVE comparison
      ↓
Risk scoring
      ↓
Finding generated
      ↓
Notification (Security Hub / SNS / EventBridge)
      ↓
Patch (manual or automated via Patch Manager)
      ↓
Automatic rescan
      ↓
Finding closed
```

---

## 17. Deployment Steps

1. Open AWS Inspector in the console.
2. Enable Inspector for the account/organization.
3. Enable EC2 scanning.
4. Enable ECR scanning.
5. Enable Lambda scanning.
6. Ensure Systems Manager (SSM Agent) is installed and healthy on EC2 instances that need scanning.
7. Review findings in the Inspector dashboard or Security Hub.
8. Patch identified vulnerabilities (manually or via Patch Manager automation).
9. Verify findings move to "Resolved" after rescanning.

---

## 18. Integrations

- **AWS Security Hub** — centralised findings and compliance view
- **Amazon EventBridge** — trigger automated remediation workflows
- **Amazon SNS** — notifications/alerts
- **AWS Systems Manager** — EC2 inventory + Patch Manager for remediation
- **AWS Organizations** — multi-account, org-wide enablement
- **Amazon CloudWatch** — metrics, dashboards, alerting
- **AWS Config** — complementary configuration compliance (not vulnerability scanning)
- **AWS CloudTrail** — audit trail of Inspector API activity

---

## 19. Cost Model

Inspector uses a **pay-for-what-you-scan** pricing model. Charges generally depend on:

- Number of EC2 instances scanned
- Number of ECR container images scanned
- Number of Lambda functions scanned
- Frequency/volume of rescanning
- Overall workload size

**Cost optimisation tips:**
- Remove unused/idle EC2 instances
- Delete stale or unused container images from ECR
- Remove unused Lambda functions
- Avoid scanning environments that don't need continuous coverage (e.g., truly ephemeral test resources)

---

## 20. Best Practices

- Enable Inspector organisation-wide (all accounts, all relevant regions)
- Enable all supported resource types (EC2, ECR, Lambda, Code Security where applicable)
- Integrate findings into Security Hub for a single pane of glass
- Automate remediation using EventBridge rules where safe to do so
- Patch Critical/High findings immediately — don't let them sit
- Keep the SSM Agent healthy and updated on EC2 instances
- Bake Inspector scanning into the CI/CD pipeline (scan images before they reach production)
- Regularly review and clean up unused resources to control cost and reduce attack surface

---

## 21. Common Mistakes

- Ignoring Critical/High findings for too long
- Not enabling Inspector in all active regions
- Forgetting to enable ECR scanning (leaving container images unchecked)
- Assuming Inspector replaces GuardDuty (it doesn't — GuardDuty is threat detection, Inspector is vulnerability management)
- Assuming Inspector scans every AWS service (it only covers EC2, ECR, Lambda, and optional code scanning)
- Treating every CVE with equal urgency instead of using Inspector's risk/context scoring to prioritise

---

## 22. Production Security Architecture

```
Internet
   ↓
Application Load Balancer
   ↓
EC2 / ECS / EKS Workloads
   ↓
Images stored in Amazon ECR
   ↓
AWS Inspector scans EC2, ECR images, Lambda
   ↓
Security Hub
   ↓
EventBridge
   ↓
Systems Manager Patch Manager
   ↓
Automatic Rescan
```

---

## 23. Related AWS Security Services (Quick Map)

Since Inspector deliberately doesn't cover every service, it's worth knowing which service covers what:

| Purpose | Service |
|---|---|
| Vulnerability Management (EC2, ECR, Lambda, Code) | **AWS Inspector** |
| Threat Detection (malicious activity, anomalies) | **Amazon GuardDuty** |
| Centralised security findings | **AWS Security Hub** |
| Configuration compliance | **AWS Config** |
| Sensitive data discovery in S3 | **Amazon Macie** |
| IAM permission analysis | **IAM Access Analyzer** |
| Central firewall policy management | **AWS Firewall Manager** |
| Network path / reachability analysis | **VPC Reachability Analyzer / Network Access Analyzer** |

---

## 24. Summary

AWS Inspector is a fully managed, **continuous vulnerability management** service focused specifically on **compute workloads** — because only workloads that run code and install software packages can meaningfully have "vulnerabilities" in the CVE sense.

**Primary supported workloads:**
- Amazon EC2
- Amazon ECR
- AWS Lambda
- Inspector Code Security (optional, source code)

**What it explicitly does not cover:** S3, IAM, RDS, DynamoDB, VPC configuration, EKS/ECS control planes — these are handled by other purpose-built AWS security services (Macie, IAM Access Analyzer, Config, Firewall Manager, etc.).

**Scanning model:** Modern Inspector is "agentless" in the sense that it removed the old, dedicated Inspector Agent requirement — but for EC2 it still depends on the **SSM Agent** for software inventory. For ECR and Lambda, it works entirely through native AWS integrations with no agent involved at all.

It automatically discovers resources, builds a software inventory, cross-references continuously updated CVE intelligence, scores risk using context like internet exposure, integrates tightly with the rest of the AWS security stack (Security Hub, EventBridge, Systems Manager), and keeps rescanning automatically until every vulnerability is remediated.
