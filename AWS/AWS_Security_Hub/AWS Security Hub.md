# AWS Security Hub 

---

## 1. What is AWS Security Hub?

AWS Security Hub is a **cloud security posture management (CSPM)** service that gives you a **single, aggregated view** of your security state across AWS accounts and services.

It does NOT scan your resources itself (mostly). Instead, it:
- **Aggregates findings** from other AWS security services (GuardDuty, Inspector, Macie, IAM Access Analyzer, Firewall Manager, etc.)
- **Runs automated compliance checks** against security standards (CIS, PCI-DSS, NIST, AWS Foundational Security Best Practices)
- **Normalizes everything** into a single finding format called **AWS Security Finding Format (ASFF)**
- **Correlates and prioritizes** findings so you're not drowning in noise from 10 different consoles

Think of it as the "central nervous system" or "SOC dashboard" for AWS-native security tooling — not a scanner itself, but the brain that collects what all the scanners find.

---

## 2. Why Security Hub Exists (The Problem It Solves)

Without Security Hub, a typical AWS security setup looks like this:

| Service | What it tells you | Where you check it |
|---|---|---|
| GuardDuty | Threats/malicious activity | GuardDuty console |
| Inspector | Vulnerabilities in EC2/ECR/Lambda | Inspector console |
| IAM Access Analyzer | Overly permissive policies | IAM console |
| Macie | Sensitive data exposure in S3 | Macie console |
| Config | Resource compliance drift | Config console |
| Firewall Manager | WAF/SG policy violations | Firewall Manager console |

That's **6 different consoles**, 6 different alert formats, no prioritization, and no single compliance score. For a security team (or a fresher trying to demonstrate security ops skill), that's unmanageable at scale.

**Security Hub's job:** pull all of it into one pane of glass, normalize the format, score it, and let you act on it — including sending it downstream to SIEM tools, ticketing systems, or automation (EventBridge → Lambda → auto-remediate).

---

## 3. Core Concepts

### 3.1 AWS Security Finding Format (ASFF)
Every finding — whether from GuardDuty, Inspector, Macie, or a third-party tool — gets normalized into a common JSON schema (ASFF). This is what makes cross-service correlation possible.

Key ASFF fields:
- `SchemaVersion`, `Id`, `ProductArn`, `GeneratorId`
- `Severity` (Label: INFORMATIONAL / LOW / MEDIUM / HIGH / CRITICAL)
- `Resources[]` (what's affected)
- `Compliance` (pass/fail against a standard control)
- `WorkflowState` (NEW / NOTIFIED / RESOLVED / SUPPRESSED)
- `RecordState` (ACTIVE / ARCHIVED)

### 3.2 Security Standards (Compliance Frameworks)
Security Hub runs **automated checks** against:
- **AWS Foundational Security Best Practices (FSBP)** — AWS's own opinionated baseline
- **CIS AWS Foundations Benchmark** (v1.2.0, v1.4.0, v3.0.0)
- **PCI DSS**
- **NIST 800-53**

Each standard = a set of **controls** (e.g., "S3 buckets should have server-side encryption enabled"). Each control maps to one or more checks run continuously.

### 3.3 Insights
Saved, reusable queries that group findings by a common attribute (e.g., "top 10 resources with the most HIGH severity findings"). AWS provides managed insights; you can create custom ones.

### 3.4 Security Score
Security Hub calculates a **percentage compliance score per standard** based on: (passed controls) / (passed + failed controls). This is the number leadership actually looks at.

### 3.5 Cross-Region Aggregation
You can designate one **aggregation Region** and Security Hub will pull findings from all your other enabled Regions into it — so you get one true global view instead of checking region by region.

### 3.6 Cross-Account Aggregation (via AWS Organizations)
Designate a **delegated administrator account**. That account can auto-enable Security Hub for all member accounts and see a consolidated view across the entire Org — critical for multi-account setups (which is basically every real enterprise AWS environment).

---

## 4. Architecture (High-Level)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/696a743c-f383-4503-98ee-7126f7de9654" />


---

## 5. How It Works — Flow

1. **Enable Security Hub** in a Region (per-account or via delegated admin for Org-wide).
2. Security Hub **auto-enables integration** with GuardDuty, Inspector, Macie, Config, IAM Access Analyzer, etc. (if those services are also enabled — Security Hub doesn't turn them on for you, it just ingests their output).
3. Choose which **security standards** to enable (FSBP is on by default for new accounts).
4. Security Hub continuously runs **automated checks** (uses AWS Config rules under the hood for many controls — so **AWS Config must be enabled** in the Region).
5. Findings flow in from all integrated services → normalized to ASFF → deduplicated → scored.
6. You view findings in the console, grouped/filtered/sorted, or query via **Insights**.
7. Findings can trigger **EventBridge rules** → Lambda/SSM Automation for auto-remediation, or forwarded to Slack/JIRA/SIEM.
8. You can update `WorkflowState` on a finding (mark as NOTIFIED, RESOLVED, or SUPPRESSED) to track remediation lifecycle.

---

## 6. Advantages

| Advantage | Detail |
|---|---|
| **Single pane of glass** | No more checking 6 consoles separately |
| **Normalized data (ASFF)** | Enables correlation and consistent tooling downstream |
| **Automated compliance scoring** | CIS/PCI/NIST scores calculated continuously, not manually audited |
| **Multi-account, multi-region aggregation** | Critical for real enterprise AWS Organizations setups |
| **No agents required** | It's a findings aggregator, not a scanner — lightweight to enable |
| **Native EventBridge integration** | Enables auto-remediation pipelines (Lambda, SSM Automation) |
| **Third-party integration support** | Ingests findings from ~75+ partner security products (Trend Micro, Palo Alto, Qualys, CrowdStrike, etc.) |
| **Reduces alert fatigue** | Deduplication + severity scoring helps prioritize what matters |
| **Continuous, not point-in-time** | Unlike a manual audit, checks run continuously as config changes |

---

## 7. Disadvantages / Limitations

| Disadvantage | Detail |
|---|---|
| **Not a scanner itself** | Depends entirely on GuardDuty/Inspector/Config/Macie being enabled and configured correctly — garbage in, garbage out |
| **Cost stacks up** | You pay for Security Hub checks AND for each underlying service (GuardDuty, Inspector, Config) separately — this surprises a lot of people |
| **Requires AWS Config** | Many controls need Config enabled in the Region, which has its own cost and setup overhead |
| **Alert volume can still be high** | Especially right after enabling FSBP — hundreds of findings can appear immediately (e.g., old resources without encryption) |
| **No remediation out-of-the-box** | It surfaces findings; you must build remediation automation yourself (EventBridge + Lambda/SSM) |
| **Cross-account setup complexity** | Delegated administrator + Organizations integration takes real planning, not a 2-click setup |
| **Some findings are noisy/low-value** | INFORMATIONAL severity findings can clutter the view if not filtered |
| **Region-scoped by default** | You must explicitly configure cross-region aggregation; it's not automatic |
| **Not real-time for all checks** | Config-based checks can lag — not instantaneous like a GuardDuty threat detection |

---

## 8. Pricing Model (High-Level)

- Charged per **security check** performed (per account, per Region) and per **finding ingested** from integrated products, after a monthly free tier.
- Costs scale with: number of accounts × number of Regions × number of enabled standards × number of resources.
- **This is the #1 real-world gotcha**: enabling all 4 standards across all Regions in a large multi-account Org can get expensive fast. Best practice — enable only the standards you actually need to comply with, and use Region/account scoping deliberately.
- (Always check the AWS Security Hub pricing page for current rates — this changes and shouldn't be memorized as a fixed number.)

---

## 9. Common Use Cases

- **Centralized security dashboard** for a SOC or DevOps team managing multiple AWS accounts
- **Continuous compliance auditing** for PCI-DSS/CIS/NIST without manual audits
- **Auto-remediation pipelines**: e.g., a finding "S3 bucket is public" → EventBridge → Lambda → auto-apply bucket policy to block public access
- **Feeding a SIEM** (Splunk, Sumo Logic) with normalized AWS security data
- **Executive reporting**: compliance score trending over time for leadership/audit purposes
- **Pre-audit readiness check** before a PCI-DSS or SOC 2 audit

---

## 10. Security Hub vs Related Services (Interview-Critical)

| Service | Purpose | Relationship to Security Hub |
|---|---|---|
| **GuardDuty** | Threat detection (malicious activity, compromised credentials, anomalous API calls) | Feeds findings INTO Security Hub |
| **Inspector** | Vulnerability scanning (EC2, ECR, Lambda — CVEs, network reachability) | Feeds findings INTO Security Hub |
| **Macie** | Sensitive data discovery in S3 (PII, credentials) | Feeds findings INTO Security Hub |
| **Config** | Tracks resource configuration + compliance rules | Powers many Security Hub compliance checks under the hood |
| **IAM Access Analyzer** | Finds overly permissive resource policies | Feeds findings INTO Security Hub |
| **Firewall Manager** | Centrally manages WAF/SG/Shield policies across accounts | Feeds findings INTO Security Hub |
| **Security Hub** | Aggregates ALL of the above + runs compliance standards | The central hub — doesn't replace any of them |

**Key interview line**: *"Security Hub doesn't detect threats or scan for vulnerabilities itself — it's the aggregation and compliance layer that sits on top of GuardDuty, Inspector, Macie, Config, and IAM Access Analyzer."*

---

## 11. Enabling Security Hub — Quick Console Steps

1. Open **Security Hub console** → Region of choice.
2. Click **Go to Security Hub** → Choose which standards to enable (FSBP recommended as baseline).
3. Confirm — Security Hub begins ingesting findings from already-enabled services (GuardDuty, Inspector, etc.) and running Config-based checks.
4. (Optional, recommended for real environments) Go to **Settings → Regions** → configure the **aggregation Region** to consolidate multi-Region findings.
5. (Optional, for Orgs) In the **management account**, go to Security Hub → **Settings → Delegated Administrator** → designate a member account → that account can now auto-enable Security Hub org-wide.
6. Review **Summary** dashboard → Security score per standard.
7. Go to **Findings** tab → filter by severity/resource type/workflow state.
8. Go to **Insights** tab → use managed insights or create custom ones.

---

## 12. Troubleshooting

| Issue | Cause | Fix |
|---|---|---|
| No findings showing up | Underlying services (GuardDuty/Inspector) not enabled | Enable the source service first — Security Hub only aggregates |
| Compliance checks missing/incomplete | AWS Config not enabled in that Region | Enable Config; many FSBP/CIS controls depend on it |
| Findings not aggregating across accounts | No delegated administrator configured | Set up Organizations integration + delegated admin |
| Findings not aggregating across Regions | No aggregation Region configured | Set aggregation Region under Settings → Regions |
| Sudden huge spike in findings after enabling | Normal — FSBP evaluates existing resources retroactively | Triage by severity, suppress/resolve as remediated |
| High cost | All standards + all Regions + all accounts enabled | Scope down to needed standards/Regions only |

---

## 13. Interview Q&A

**Q: What is AWS Security Hub in one line?**
A: A CSPM service that aggregates and normalizes security findings from AWS-native and third-party services into one dashboard, and runs automated compliance checks against standards like CIS, PCI-DSS, and NIST.

**Q: Does Security Hub replace GuardDuty or Inspector?**
A: No. It aggregates their findings. You still need GuardDuty for threat detection and Inspector for vulnerability scanning — Security Hub is the layer on top.

**Q: What format does Security Hub use to normalize findings?**
A: AWS Security Finding Format (ASFF).

**Q: How does Security Hub achieve multi-account visibility?**
A: Through AWS Organizations, by designating a delegated administrator account that aggregates findings across all member accounts.

**Q: What AWS service do many Security Hub compliance checks depend on?**
A: AWS Config — it evaluates resource configuration against rules that back many FSBP/CIS controls.

**Q: What's the difference between WorkflowState and RecordState in a finding?**
A: WorkflowState tracks your remediation progress (NEW/NOTIFIED/RESOLVED/SUPPRESSED); RecordState tracks whether the finding is still relevant (ACTIVE/ARCHIVED, e.g., auto-archived if the underlying resource is deleted).

**Q: How would you auto-remediate a Security Hub finding?**
A: Route the finding via an EventBridge rule (matching on finding type/severity) to a Lambda function or SSM Automation runbook that performs the fix (e.g., block public S3 access).

**Q: What's a real cost pitfall with Security Hub?**
A: Enabling all standards across all Regions/accounts without scoping — cost scales with checks performed and findings ingested, so it's easy to over-provision unnecessarily.

---

