# AWS CSPM (Cloud Security Posture Management) — End to End

> **Verified:** October 4, 2026 against AWS documentation and the AWS pricing page (sources in section 16).
> **Scope:** concepts, how AWS Security Hub CSPM works, setup, multi-account design, prioritization, remediation, billing, operations, pitfalls.
> Standards, control IDs, and prices change. Re-check the linked AWS pages before production decisions.

---

## 1. What is CSPM?

**Cloud Security Posture Management** continuously checks cloud **configuration** (control plane) against security best practices and compliance standards, and helps you prioritize and fix drift.

Why it matters: many cloud incidents come from misconfiguration (public storage, open ports, over-permissive IAM, unencrypted data) rather than exotic exploits. Firewalls, WAFs, and SIEMs watch traffic and logs; they don't tell you a resource is configured insecurely.

### CSPM vs Vulnerability Management

| | CSPM | Vulnerability Management |
|---|---|---|
| Target | Cloud resource configuration | Software, packages, images, code |
| Finds | Public S3, open security groups, weak IAM, unencrypted databases | CVEs, missing patches, vulnerable dependencies |
| Goal | Continuous compliance, baseline hygiene | Patch/fix flaws before exploitation |
| AWS service | Security Hub CSPM (+ AWS Config) | Amazon Inspector |

They complement each other: a CVE on an internal-only host is lower risk; the same CVE on a host exposed by a misconfigured security group is critical.

---

## 2. AWS CSPM building blocks

### Naming (important)
- **AWS Security Hub CSPM** is the service formerly called AWS Security Hub: security best-practice checks, finding aggregation, compliance monitoring.
- **AWS Security Hub** (new) is a unified cloud security solution that correlates signals from CSPM, Amazon Inspector, GuardDuty, Macie and others, with findings in **OCSF** format. CSPM is a component of it and is **billed separately** based on its own usage.
- This guide covers **Security Hub CSPM**. Findings in CSPM use **ASFF** (AWS Security Finding Format).
- CSPM can also evaluate **Microsoft Azure** resources once you set up an integration (same pricing dimensions, own free trial).

| Component | Role |
|---|---|
| **AWS Security Hub CSPM** | Runs security checks, calculates security score, aggregates findings, drives workflows |
| **AWS Config** | Records resource configuration; most CSPM controls run on Config rules (service-linked rules) |
| **Amazon GuardDuty** | Threat detection findings (feed into Security Hub) |
| **Amazon Inspector** | Vulnerability findings |
| **Amazon Macie** | Sensitive data findings for S3 |
| **IAM Access Analyzer, Firewall Manager** | Additional finding sources |
| **Amazon EventBridge** | Routes findings to automation targets |
| **Lambda / SSM Automation / Step Functions** | Remediation execution |
| **AWS Organizations** | Multi-account management via a delegated administrator |

### AWS Config dependency
- Most controls are evaluated through AWS Config rules that Security Hub CSPM creates for you (**service-linked rules**).
- You must turn on **AWS Config resource recording** for the resource types behind your enabled controls. This is mainly needed for **change-triggered** controls, and some periodic controls also need it. AWS recommends enabling recording **before** you enable standards.
- If Config is off, the control **Config.1** raises FAILED findings, and other findings can be inaccurate or stale.
- Security Hub CSPM controls evaluate resource configuration directly and **do not take AWS Organizations policies (such as SCPs) into account**.

---

## 3. How it works (end-to-end flow)

```
Resources (EC2, S3, IAM, VPC, RDS...)
        │  configuration changes
        ▼
   AWS Config (recorder + service-linked rules)
        │  evaluations
        ▼
Security Hub CSPM  ◄── GuardDuty / Inspector / Macie / Access Analyzer / partners
   • runs controls from enabled standards (security checks)
   • generates findings (ASFF)
   • calculates security score
        │
        ├─► Automation rules (update / suppress findings)
        ├─► Insights & dashboards (prioritize)
        ├─► EventBridge ──► Lambda / SSM / Step Functions (remediate)
        └─► Integrations: ticketing, chat, SIEM/SOAR
        │
        ▼
 Re-evaluation → PASSED → workflow status RESOLVED (automatic)
```

### The 5-step process
1. **Discover** — Config records the resources in each account/Region.
2. **Assess** — Controls evaluate configuration against enabled standards, per resource.
3. **Prioritize** — Severity, insights, automation rules.
4. **Remediate** — Manual (guided steps) or automated (EventBridge → Lambda/SSM).
5. **Monitor & prove** — Continuous re-evaluation, finding history, audit evidence.

### When checks run
- After you enable a standard, checks begin **within about two hours**. Controls that share a Config rule with an already-enabled standard can take **up to 24 hours** to produce findings.
- **Periodic** checks run within 12 or 24 hours of the previous run (AWS decides; you can't change it).
- **Change-triggered** checks run when the resource changes state. With Config *daily* recording, findings can lag until the 24-hour delivery completes. Security Hub CSPM also runs a catch-up check at least daily.
- Plan for **detection delay**: CSPM is near-continuous, not instantaneous.

---

## 4. Standards and controls

A **standard** is a set of requirements; Security Hub CSPM maps them to **controls** and runs **security checks**. Supported standards include:

| Standard | Notes |
|---|---|
| **AWS Foundational Security Best Practices (FSBP)** | AWS's own baseline (v1.0.0); common starting point |
| **CIS AWS Foundations Benchmark** | Versions **5.0.0, 3.0.0, 1.4.0, 1.2.0** supported |
| **NIST SP 800-53 Rev. 5** | US federal-aligned controls |
| **NIST SP 800-171 Rev. 2** | Protecting Controlled Unclassified Information |
| **PCI DSS** | Payment card environments (check supported versions in the console/docs) |
| **AWS Resource Tagging** | Tag hygiene |
| **Service-managed standard: AWS Control Tower** | Detective controls configured from Control Tower |

AWS states that standards and controls **don't guarantee compliance** with any framework or audit. Treat results as evidence, not certification.

Tips:
- Start with **FSBP**; add others only for a real compliance driver.
- **Billing note:** if identical controls appear in multiple standards and evaluate the same resource, you are charged **once** for that check.
- Disable controls that don't apply, with a recorded reason.

### Key finding fields (ASFF)
- **Compliance.Status:** `PASSED`, `FAILED`, `WARNING`, `NOT_AVAILABLE`
- **Severity label:** `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `INFORMATIONAL`
- **Workflow.Status:** `NEW`, `NOTIFIED`, `SUPPRESSED`, `RESOLVED`
- **RecordState:** `ACTIVE`, `ARCHIVED`
- **Compliance.SecurityControlId** (e.g., `S3.2`) — AWS recommends filtering on this (or the control ARN), not on title or description.

### Workflow status behavior (control findings)
- `PASSED` **automatically** sets `Workflow.Status` to `RESOLVED`.
- If a finding goes from `PASSED` back to `FAILED`/`WARNING`/`NOT_AVAILABLE` while `NOTIFIED` or `RESOLVED`, it is reset to `NEW`.
- Manually setting `SUPPRESSED` or `RESOLVED` **does not stop** Security Hub CSPM from generating a new finding for the same problem. Fix the cause or disable the control.
- If AWS Config returns NOT_APPLICABLE, the finding is archived.
- When computing a control's overall status, archived and suppressed findings are ignored.

### Security score
Percentage-based score derived from passed vs. failed checks across enabled standards and controls. Use it as a **trend metric**: disabling controls can raise the score without reducing risk.

---

## 5. Setup — single account

### Console
1. Enable **AWS Config** and turn on recording for the resource types your controls need (recommended: all supported resource types; include global resources in one Region).
2. Open **Security Hub CSPM → Enable**, then choose standards (start with FSBP).
3. Wait for the first evaluations (see "When checks run" above).
4. Review **Controls**, **Findings**, **Insights**.

---

## 6. Multi-account / multi-Region design

Security Hub CSPM is **regional**: enable it in each Region you use.

1. **Delegated administrator** — the Organizations **management account** designates one account as the Security Hub CSPM delegated administrator (this is also the Security Hub CSPM administrator account). Use a dedicated security account.
2. **Central configuration (optional, recommended at scale)** — the delegated admin creates **configuration policies** that enable/disable Security Hub CSPM, standards, and controls across accounts, OUs, or the whole organization.
   - You specify a **home Region** and **linked Regions**, and policies are managed from the home Region.
   - AWS offers a recommended policy; you can create up to 20 custom policies.
   - Central configuration **does not manage AWS Config recorder settings**; enable Config separately.
   - With central configuration, controls that involve global resources are automatically disabled in all Regions except the home Region.
   - A policy that disables Security Hub CSPM can't be associated with the delegated admin account.
3. **Local configuration (alternative)** — the delegated admin can auto-enable Security Hub CSPM and a limited set of standards for new accounts in the current Region.
4. **Cross-Region aggregation** — replicates findings, finding updates, insights, control compliance statuses, and security scores from linked Regions to the **home Region**. Updates made in the home Region replicate back. It adds **no cost** and only aggregates Regions where CSPM is enabled. Linking modes: `ALL_REGIONS`, `ALL_REGIONS_EXCEPT_SPECIFIED`, `SPECIFIED_REGIONS`, `NO_REGIONS`. Cross-Region aggregation is a prerequisite for central configuration.
5. Controls still run per account and per Region; aggregation only gives you one view.

---

## 7. Prioritization

- **Insights**: groupings of findings (managed and custom) to highlight what matters.
- **Automation rules** update or suppress findings automatically, in near real time.
  - Created and managed from the **administrator account** only.
  - Maximum **100 rules** per administrator account; rules are **per Region** (create in each Region).
  - Each rule has criteria and actions and an **order**; rules apply in ascending order.
  - Rules apply to **new and updated** findings after the rule is created.
  - Typical uses: raise severity for production accounts, suppress known-accepted findings in sandboxes, add notes, set workflow status.
- Pair with **tags and account ownership** so findings reach the right team.

Suggested triage order: Critical/High on internet-exposed, production, sensitive-data resources first; then by severity and age.

---

## 8. Remediation

### 8.1 Manual
Each control has remediation guidance. Fix the resource, wait for re-evaluation; the finding becomes `PASSED` and workflow `RESOLVED` automatically.

### 8.2 Automated (event-driven)

```
Security Hub finding ──► EventBridge rule ──► Lambda / SSM Automation / Step Functions ──► fix resource
```

Two trigger styles:
- **Custom action** — an analyst selects findings and sends them to EventBridge (event type `Security Hub Findings - Custom Action`). Human in the loop.
- **Automatic** — an EventBridge rule on event type `Security Hub Findings - Imported`, source `aws.securityhub`.

Notes:
- `Imported` events fire for new findings **and for updates**, including updates you make with `BatchUpdateFindings`. Filter tightly so your own updates don't re-trigger the rule.
- Filter control findings by `SecurityControlId`, not title/description.

#### Example EventBridge pattern (S3 public access controls)
```json
{
  "source": ["aws.securityhub"],
  "detail-type": ["Security Hub Findings - Imported"],
  "detail": {
    "findings": {
      "Compliance": {
        "Status": ["FAILED"],
        "SecurityControlId": ["S3.2", "S3.8"]
      },
      "Workflow": { "Status": ["NEW"] },
      "RecordState": ["ACTIVE"],
      "Resources": { "Type": ["AwsS3Bucket"] }
    }
  }
}
```
Control IDs used above: **S3.2** (general purpose buckets should block public read access) and **S3.8** (general purpose buckets should block public access). Verify IDs in the current controls reference before deploying.

#### Example Lambda (enable S3 Block Public Access on the bucket)
```python
import boto3

s3 = boto3.client("s3")
sh = boto3.client("securityhub")

def handler(event, context):
    for f in event["detail"]["findings"]:
        for r in f["Resources"]:
            if r["Type"] != "AwsS3Bucket":
                continue
            bucket = r["Id"].split(":::")[-1]
            s3.put_public_access_block(
                Bucket=bucket,
                PublicAccessBlockConfiguration={
                    "BlockPublicAcls": True,
                    "IgnorePublicAcls": True,
                    "BlockPublicPolicy": True,
                    "RestrictPublicBuckets": True,
                },
            )
            # Record what happened. Do NOT set RESOLVED manually:
            # Security Hub sets RESOLVED itself when the control re-evaluates to PASSED.
            sh.batch_update_findings(
                FindingIdentifiers=[{"Id": f["Id"], "ProductArn": f["ProductArn"]}],
                Workflow={"Status": "NOTIFIED"},
                Note={"Text": "Auto-remediation: S3 Block Public Access enabled",
                      "UpdatedBy": "auto-remediation"},
            )
```
Warnings:
- Blocking public access **will break** buckets that are intentionally public (e.g., static websites). Exempt them with a tag check before remediating.
- Lambda role needs least privilege: `s3:PutBucketPublicAccessBlock` on target buckets, `securityhub:BatchUpdateFindings`, and CloudWatch Logs permissions.
- In multi-account setups, run remediation in the **workload account** (cross-account event routing + a role there) instead of giving the central account broad write access.

### 8.3 Prebuilt option: Automated Security Response on AWS
AWS provides **Automated Security Response on AWS (ASR)** — playbooks of predefined remediations for Security Hub findings, deployed with CloudFormation.
- Playbooks cover FSBP v1.0.0, CIS (1.2.0, 1.4.0, 3.0.0), PCI DSS v3.2.1, NIST, and a **Security Control** playbook.
- If **consolidated control findings** are enabled, the **Security Control** playbook is the only one that should be enabled.
- Automated remediations are toggled **per control** and are **disabled by default**; you can still invoke a remediation manually from the console.

### 8.4 Safe-automation rules
- Automate only **low-blast-radius, well-understood** fixes (e.g., Block Public Access, enabling encryption defaults, enabling logging).
- Don't auto-fix changes that can break workloads (closing security group ports apps depend on, removing IAM users/keys) without approval.
- Support exceptions via tags and log every action.
- Test in non-production; add a dead-letter queue and alarms on the remediation function.
- Prevent recurrence at the source (SCPs, IaC policy checks, secure defaults). Remember CSPM controls **don't consider SCPs**, so a resource blocked by an SCP may still be reported on its configuration.

---

## 9. Integrations

- **Ticketing and chat** (e.g., Jira, Slack, Microsoft Teams) via partner integrations or EventBridge.
- **SIEM/SOAR** via EventBridge or partner integrations.
- **New AWS Security Hub** receives CSPM findings and normalizes findings in OCSF.
- Security Hub CSPM sends findings to EventBridge automatically; targets can be Lambda, Step Functions, or Systems Manager Automation runbooks.

---

## 10. Operating model (suggested)

| Cadence | Activity |
|---|---|
| Continuous | Automated remediation for safe controls; alerts for Critical/High |
| Daily | Triage new Critical/High findings, assign owners |
| Weekly | Review trends, aging findings, suppression requests |
| Monthly | Score trend, top failing controls, exception review |
| Quarterly | Standards review, control tuning, remediation playbook tests, audit evidence pull |

Metrics: open Critical/High count, mean time to remediate, % findings auto-remediated, aging beyond SLA, number of exceptions, coverage (accounts/Regions with Config + Security Hub CSPM enabled).

---

## 11. Billing variables and cost model

Security Hub CSPM is priced on **three dimensions** per month: **security checks**, **finding ingestion events**, and **automation rule evaluations**. With AWS Organizations, usage across accounts is consolidated for **tiered pricing**. **AWS Config is billed separately.**

### 11.1 Variables

| Variable | Meaning | Billing rule |
|---|---|---|
| `A` | Accounts with CSPM enabled | Multiplies usage |
| `R` | Regions enabled | Costs computed per Region |
| `C` | **Security checks**: one control evaluated against one resource | Tiered; identical controls shared across standards are charged **once** per resource |
| `F` | **Finding ingestion events**: new findings **and updates** from AWS services, partners, Azure | **Free** for events from CSPM's own security checks; perpetual free tier of **10,000 events/month** |
| `N_rules`, `N_crit` | Number of automation rules, criteria per rule | Drive `E` |
| `E` | **Automation rule evaluations** = `(C + F) × N_rules × N_crit` (per AWS's pricing examples) | **First 1,000,000/month free**; tiered after |
| `CI` | AWS Config **configuration items** recorded | **Separate** AWS Config pricing |
| Config rules | Rules Security Hub CSPM enables (service-linked) | **Not charged separately** |
| Trial | 30-day free trial per account per Region | Shows an estimated monthly bill during the trial |
| Extras | Remediation compute (Lambda, SSM, Step Functions, EventBridge), SIEM/export, ticketing | Billed by those services |
| Azure checks | Only if you integrate Azure | Same dimensions; Azure check rate differs |

Cross-Region aggregation does **not** add cost.

### 11.2 Formula

```
Monthly cost =
  Σ over Regions r:
     price_checks(C_r)
   + price_ingest( max(F_r − 10,000, 0) )
   + price_rules( max(E_r − 1,000,000, 0) )
  + AWS Config cost (configuration items)
  + remediation / integration costs
```

### 11.3 Unit rates shown on the AWS pricing page (examples; confirm for your Region)

| Dimension | Rate shown |
|---|---|
| Security checks (AWS) | $0.0010 per check for the first 100,000; $0.0008 per check for the next 400,000 |
| Security checks (Azure) | $0.0012 per check (first 100,000 Azure checks) |
| Finding ingestion | $0.00 for first 10,000 events; $0.00003 per event above that |
| Automation rule evaluations | First 1,000,000 free; then $0.10 per million, with lower rates ($0.05, then $0.015 per million) at higher volumes |

### 11.4 Worked example (AWS's "large organization" example)

2 Regions, 20 accounts, 500 checks/account, 10,000 ingestions/account, 30 automation rules × 5 criteria.

| Item | Calculation | Cost |
|---|---|---|
| Security checks | 500 × 20 = 10,000 checks × $0.0010 = $10.00 per Region × 2 | $20.00 |
| Finding ingestion | 10,000 × 20 = 200,000; (200,000 − 10,000) × $0.00003 = $5.70 per Region × 2 | $11.40 |
| Automation rules | (500 + 10,000) × 20 × 30 × 5 = 31,500,000; (31.5M − 1M) × $0.10/M = $3.05 per Region × 2 | $6.10 |
| **Total (CSPM only)** | | **$37.50 / month** |

AWS Config costs are **not** included in this total.

### 11.5 Cost levers

| Lever | Effect | Variable |
|---|---|---|
| Enable only needed standards | Fewer unique checks (overlapping controls are billed once) | `C` |
| Disable inapplicable controls (with reason) | Fewer checks | `C` |
| Don't enable unused Regions (restrict with SCPs) | Linear reduction | `R` |
| Reduce noisy third-party findings and finding churn | Lower ingestion | `F` |
| Fewer automation rules/criteria | Lower evaluations | `N_rules`, `N_crit` |
| Tune Config recording where risk is accepted | Fewer configuration items | `CI` |
| Consolidate under AWS Organizations | Volume tiering | tiers |

### 11.6 Where bills surprise people
- **AWS Config configuration items** (billed separately) can exceed the Security Hub CSPM charge in accounts with fast-changing resources.
- Many standards enabled in every Region "just in case" (more resources × controls evaluated).
- High-churn partner integrations generating constant finding updates.
- Automation-rule evaluations grow with `(checks + findings) × rules × criteria`.

### 11.7 Monitoring cost
- Cost Explorer filtered by service (Security Hub, Config), grouped by linked account and Region.
- AWS Budgets and Cost Anomaly Detection for Config spikes.
- Use the AWS Pricing Calculator and the free-trial estimate for forecasts.

---

## 12. Common pitfalls

1. **Config not enabled or not recording needed resource types** → missing, stale, or inaccurate findings.
2. **Enabling every standard** → noise and overlapping work.
3. **Optimizing the score** instead of risk (disabling controls to look green).
4. **No ownership mapping** → findings nobody fixes. Use tags and account ownership.
5. **Marking findings RESOLVED/SUPPRESSED by hand** and expecting them to stay closed — new findings are still generated if the issue persists.
6. **Over-automation** → remediation that causes outages (e.g., breaking intentionally public buckets).
7. **EventBridge loops** — updates via `BatchUpdateFindings` produce new `Imported` events; filter by status.
8. **Central account with excessive remediation rights** → high-value target.
9. **Ignoring Regions** → unmonitored Regions are blind spots.
10. **Expecting SCPs to change CSPM results** — controls evaluate resource configuration directly.
11. **Treating CSPM as complete security** — pair it with GuardDuty, Inspector, logging, and IAM hygiene.
12. **Unreviewed suppressions** → stale exceptions hide real exposure.

---

## 13. Limitations of CSPM

- Checks configuration state, not runtime behavior or data content.
- Control coverage varies by service and Region; some risks need custom Config rules or custom findings.
- Not instantaneous: periodic checks run every 12–24 hours, and change-triggered checks depend on Config recording.
- Doesn't guarantee compliance with any framework or audit.
- Limited business context for prioritization; correlate with exposure, identity, and data sensitivity (the newer unified Security Hub adds correlation).

---

## 14. Quick-start checklist

- [ ] Turn on AWS Config recording (needed resource types, all active Regions) before enabling standards
- [ ] Designate a delegated administrator from the Organizations management account
- [ ] Enable Security Hub CSPM with FSBP in each used Region
- [ ] Set a home Region and enable cross-Region aggregation (required for central configuration)
- [ ] Define central configuration policies (standards, disabled controls with reasons) or local auto-enable
- [ ] Create automation rules in each Region (severity bump for prod, suppress sandbox noise)
- [ ] Integrate ticketing/chat; route by account or tag owner
- [ ] Implement 2–3 low-risk auto-remediations (or evaluate Automated Security Response on AWS)
- [ ] Add preventive guardrails (SCPs, IaC scanning in CI)
- [ ] Set SLAs per severity and a review cadence
- [ ] Define the exception process and review it quarterly
- [ ] Set up Budgets/Cost Explorer views for Security Hub and Config

---

## 15. Glossary

- **ASFF** — AWS Security Finding Format, the schema for Security Hub CSPM findings.
- **OCSF** — Open Cybersecurity Schema Framework, used by the newer AWS Security Hub.
- **Control** — a single check requirement (e.g., "S3 general purpose buckets should block public read access").
- **Security check** — one control evaluated against one resource; produces a finding and is the billing unit.
- **Standard** — a collection of controls mapped to a framework.
- **Insight** — grouped view of findings.
- **Automation rule** — rule that updates or suppresses findings as they arrive.
- **Custom action** — manual trigger that sends findings to EventBridge.
- **Delegated administrator** — member account that manages Security Hub CSPM for the organization.
- **Central configuration** — policy-based management of CSPM across accounts, OUs, and Regions.
- **Home Region** — Region that aggregates findings from linked Regions (formerly "aggregation Region").
- **Service-linked rule** — AWS Config rule created by Security Hub CSPM; not charged separately.
- **Drift** — a resource deviating from its secure baseline.

---

## 16. Verification and sources

Facts in this guide were checked on October 4, 2026 against:

- Pricing: https://aws.amazon.com/security-hub/cspm/pricing/
- Features: https://aws.amazon.com/security-hub/features/ and https://aws.amazon.com/security-hub/cspm/features
- Standards: https://docs.aws.amazon.com/securityhub/latest/userguide/standards-available.html
- FSBP: https://docs.aws.amazon.com/securityhub/latest/userguide/fsbp-standard.html
- CIS versions: https://docs.aws.amazon.com/securityhub/latest/userguide/cis-aws-foundations-benchmark.html
- S3 controls: https://docs.aws.amazon.com/securityhub/latest/userguide/s3-controls.html
- AWS Config prerequisites: https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-setup-prereqs.html
- Check schedules: https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-standards-schedule.html
- Compliance and workflow status: https://docs.aws.amazon.com/securityhub/latest/userguide/controls-overall-status.html
- EventBridge rules: https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-cwe-all-findings.html
- Configuration policies: https://docs.aws.amazon.com/securityhub/latest/userguide/configuration-policies-overview.html
- Cross-Region aggregation: https://docs.aws.amazon.com/securityhub/latest/userguide/finding-aggregation.html
- Automation rules: https://docs.aws.amazon.com/securityhub/latest/userguide/automation-rules.html
- Automated Security Response on AWS: https://docs.aws.amazon.com/solutions/latest/automated-security-response-on-aws/playbooks.html

Not independently confirmed from AWS sources, and therefore worded generally: supported PCI DSS versions, tier boundaries for automation-rule pricing above the first million evaluations, and the exact catch-up check interval (AWS docs say at least daily).
