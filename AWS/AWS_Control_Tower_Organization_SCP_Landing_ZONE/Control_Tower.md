# AWS Control Tower — Easy Guide

> Learn Control Tower from zero, in simple words, without losing the important technical details.

---

## 1. What is AWS Control Tower?

**AWS Control Tower** is an AWS service that helps organizations set up and govern a **multi-account AWS environment** using AWS best practices.

In simple words:

> Control Tower helps you build a standard AWS environment and keep many AWS accounts under common governance.

It does **not** replace every underlying AWS service — it **orchestrates** several AWS services and capabilities to create and operate a governed environment.

```
                 AWS Control Tower
                        |
        -----------------------------------
        |                |                |
     Setup            Governance       Visibility
        |                |                |
  Landing Zone        Controls        Dashboard
        |
  Account Factory
        |
  Standard Accounts
```

**One-line definition:**
Control Tower = centralized setup + governance + account provisioning + compliance visibility for a multi-account AWS environment.

---

## 2. Why do we need it?

Imagine a company has 100 AWS accounts, each configured differently:

```
Account 1  → different configuration
Account 2  → different security
Account 3  → different logging
Account 4  → different policies
...
Account 100
```

This creates:
- Security gaps
- Configuration differences
- Compliance problems
- Difficult auditing
- Manual account setup
- Configuration drift

Control Tower creates a **consistent baseline** and gives **centralized governance**, so every governed account:
- ✓ follows required security controls
- ✓ has required logging
- ✓ follows company governance
- ✓ is monitored for compliance

---

## 3. When should you use it?

Useful when:
- You have multiple AWS accounts
- You're building a new multi-account environment
- You need centralized governance
- Security/compliance matters
- You need standardized account provisioning
- Many teams need separate accounts
- You want continuous governance visibility

**When it may not be necessary:** if you only have one or two accounts with no real multi-account governance need — Control Tower may add more complexity than value.

---

## 4. The big picture

```
                       AWS CONTROL TOWER
                              |
              ┌───────────────┼────────────────┐
              |               |                |
              ↓               ↓                ↓
         Landing Zone      Controls        Dashboard
              |
              ↓
       Account Factory
              |
              ↓
       Standard Accounts
              |
              ↓
      Continuous Governance
```

- **Control Tower** = the central service/capability.
- **Landing Zone** = the governed multi-account environment it creates and manages.

---

## 5. Main capabilities

| Capability | Simple meaning |
|---|---|
| Landing Zone | Governed multi-account foundation |
| Controls | Rules used for ongoing governance |
| Account Factory | Standardized account provisioning |
| Dashboard | Central governance/compliance visibility |
| Account enrollment | Bring existing accounts under governance |
| Drift detection/remediation | Detect when managed config changes unexpectedly |

---

## 6. Landing Zone

A **Landing Zone** is a well-architected, multi-account AWS environment based on security and compliance best practices. It's the main environment that Control Tower manages — a real, configured environment, not just a template.

**During setup, Control Tower can establish:**
- The required organizational structure
- Shared security/logging accounts
- Identity/access configuration (depending on your setup)
- Mandatory controls
- Governance configuration

AWS describes the landing zone as the **enterprise-wide container** for the accounts, OUs, users, and resources subject to governance.

> ⚠️ Important: The Landing Zone is the foundation — it doesn't mean every future workload is automatically created inside it.

---

## 7. Account Factory

**Account Factory** is Control Tower's standardized account provisioning capability.

```
Need new AWS account
        ↓
Account Factory
        ↓
Standardized account provisioning
```

**Why?** Without it, every account could be created differently. Account Factory provides:
- Standard account configuration
- Approved account parameters
- Placement in the required org structure
- Required Control Tower governance

**Example:** A company wants a `Payment-Dev` account → the admin provisions it through Account Factory → it's created/enrolled according to the configured process.

**Key distinction:**
- **Landing Zone** → the prepared/governed environment
- **Account Factory** → provisions new accounts *into* that environment

---

## 8. Controls

A **Control** is a high-level governance rule that provides ongoing governance for the AWS environment. (AWS used to call this a **Guardrail** — the term is transitioning to "Control.")

**Example:** "Public S3 access must not be allowed." → A relevant control enforces or detects this, depending on its type.

Controls are generally applied at the **OU level**, affecting all accounts inside that OU:

```
Control → OU → Accounts
```

---

## 9. Types of Controls

### 9.1 Preventive
Stops an action before it happens.
```
User tries action → Preventive Control → BLOCK
```
**Remember:** Stop it before it happens.

### 9.2 Detective
Detects a violation or non-compliant configuration after the fact.
```
Resource/config → Detective Control → Check → Compliant / Non-compliant
```
**Remember:** Find the problem.

### 9.3 Proactive
Checks resources **before** they're created (where supported).
```
Resource request → Proactive Control → Check → Pass / Fail
```
**Remember:** Check before deployment.

---

## 10. Control guidance categories

| Guidance | Easy meaning |
|---|---|
| Mandatory | Required as part of the Control Tower baseline |
| Strongly recommended | AWS strongly recommends enabling it |
| Elective | Optional, based on your organization's needs |

> Control **type** (preventive/detective/proactive) and **guidance category** (mandatory/recommended/elective) are two different properties — don't confuse them.

---

## 11. Dashboard

Gives centralized visibility into the governed environment, instead of checking every account manually. Shows:
- Provisioned accounts
- Enabled controls
- Non-compliant resources
- Compliance status
- Accounts/OUs affected by governance

```
                Control Tower Dashboard
                         |
          ┌──────────────┼──────────────┐
          ↓              ↓              ↓
       Accounts       Controls       Compliance
```

---

## 12. Account Enrollment

Control Tower isn't only for brand-new accounts — you can **enroll existing accounts** too (subject to prerequisites and a supported process).

**Why?** A company with 50 existing accounts adopting Control Tower doesn't want to abandon them — eligible accounts can be brought under governance.

> ⚠️ Enrollment has prerequisites and needs careful planning — it's not a magic button.

---

## 13. Drift

**Drift** = the environment has moved away from the configuration Control Tower expects/manages.

```
Control Tower expects: Configuration A
Someone changes it manually: Configuration B
→ Environment may now be "drifted"
```

If Control Tower–managed resources are changed manually outside supported methods, the landing zone can enter an unknown/unhealthy state.

**Remember:** Drift = actual environment no longer matches the expected managed configuration.

---

## 14. Simplified architecture

```
                         AWS CONTROL TOWER
                                |
        ┌───────────────────────┼───────────────────────┐
        |                       |                       |
        ↓                       ↓                       ↓
  Landing Zone              Controls               Dashboard
        |                       |                       |
        ↓                       ↓                       ↓
   Multi-account          Governance rules        Compliance view
   environment
        |
        ↓
  Account Factory
        |
        ↓
  New AWS Accounts
```

---

## 15. What happens when you create a Landing Zone?

```
Start Landing Zone setup
        ↓
Establish foundational organization structure
        ↓
Set up shared security/logging accounts
        ↓
Configure identity/access according to your setup
        ↓
Apply mandatory controls
        ↓
Create/manage required governance resources
        ↓
Landing Zone ready
```

Initial setup includes: creating the **Security OU**, optionally the **Sandbox OU**, creating/adding the **Log Archive** and **Audit** shared accounts, setting up the identity configuration, and applying mandatory controls.

---

## 16. Shared accounts

**Management account**
- Manages the landing zone
- Handles Account Factory provisioning
- Manages OUs and controls
- Billing for the landing zone
- ⚠️ Best practice: don't use it for normal production workloads

**Log Archive account** — central repository for logs from all accounts:
```
Dev ───────┐
Test ──────┤
Prod ──────┼──→ Log Archive
Security ──┘
```

**Audit account** — restricted account for security/compliance teams to audit and investigate.

---

## 17. How a new account flows through Control Tower

```
Developer / Admin requests account
              ↓
        Account Factory
              ↓
        Account creation
              ↓
     Account configuration
              ↓
     Organizational placement
              ↓
     Control Tower governance
              ↓
       Controls applied
              ↓
       Account available
```

---

## 18. Real-world scenario: large company

A company with 500 accounts across Development, Testing, Production, Data, Security, and Sandbox teams faces (without governance): different configs, different security, manual creation, compliance gaps, difficult auditing, drift.

**With Control Tower:**
```
                Control Tower
                      |
                Landing Zone
                      |
        ┌─────────────┼─────────────┐
        ↓             ↓             ↓
      Controls     Account       Dashboard
                    Factory
                      |
                      ↓
                 New Accounts
```

Now: accounts provision consistently, governance applies across OUs, compliance is monitored, existing accounts can be enrolled, and admins get centralized visibility.

---

## 19. Scenario: new development account

```
Business Request
       ↓
Account Factory
       ↓
Payment-Dev Account
       ↓
Configured according to Control Tower process
       ↓
Governance / Controls
       ↓
Developer starts workload
```

Far more standardized than manually configuring every account from scratch.

---

## 20. Scenario: compliance requirement

```
Apply governance requirement
          ↓
Monitor affected accounts
          ↓
Detect non-compliance
          ↓
Administrator investigates/remediates
```

This is one of the main reasons enterprises adopt Control Tower.

---

## 21. Advantages

1. **Centralized governance** — govern many accounts from one place
2. **Standardization** — consistent account provisioning process
3. **Better security posture** — mandatory + optional controls
4. **Continuous visibility** — dashboard shows governance status
5. **Faster account provisioning** — Account Factory cuts manual setup
6. **Enterprise scalability** — built for governance across many accounts
7. **Drift awareness** — helps identify configuration deviation
8. **AWS best-practice foundation** — landing zone built on proven patterns

---

## 22. Disadvantages / limitations

1. **Additional complexity** — another governance layer to understand (controls, account lifecycle, landing zone behavior, underlying services)
2. **Learning curve** — many concepts and dependencies
3. **Not everything is automatic** — you may still need custom security, networking, monitoring, compliance, automation
4. **Customization needs planning** — enterprise needs often differ from defaults
5. **Don't casually modify managed resources** — unsupported changes cause drift or unhealthy states
6. **Cost comes from underlying services** — Control Tower itself is free, but what it enables isn't
7. **Existing environments need careful migration** — enrollment requires planning and prerequisites

---

## 23. Billing / Cost

**Most important fact:** AWS Control Tower itself has **no additional charge**. You pay for the AWS services/resources it enables or that you use in the landing zone.

| Cost area | Why it can cost money |
|---|---|
| AWS CloudTrail | API/event tracking and related usage |
| AWS Config | Configuration recording/evaluations |
| AWS Service Catalog | Used by Account Factory workflows |
| S3 | Log/configuration storage |
| KMS | Key usage where applicable |
| Lambda | Functions used by relevant workflows/controls |
| CloudWatch | Logs/metrics where used |
| VPC/NAT Gateway | Network resources created by some configurations |

```
Control Tower → CloudTrail → CloudTrail charges
```
There's no separate "Control Tower license fee."

> ⚠️ AWS specifically notes that **ephemeral workloads** in Control Tower accounts can increase AWS Config costs.

So don't ask only *"How much does Control Tower cost?"* — ask *"Which AWS services will Control Tower enable or cause us to use, and what will their usage cost?"*

---

## 24. Cost example

```
100 AWS accounts
+ CloudTrail
+ AWS Config
+ S3 logging
+ KMS
+ CloudWatch
+ networking
= Your bill (driven by these services, not Control Tower itself)
```

```
Control Tower
     |
     ├── CloudTrail ──→ Cost
     ├── Config ──────→ Cost
     ├── S3 ──────────→ Cost
     ├── KMS ─────────→ Cost
     └── Other services → Cost

Control Tower service charge → $0 additional
```

Always check current pricing for underlying services before estimating an enterprise deployment.

---

## 25. What Control Tower does NOT mean

**Control Tower ≠ Complete AWS security solution.**

It provides a **governance foundation**. You may still need other AWS security, networking, monitoring, backup, cost-management, and compliance capabilities.

---

## 26. Control Tower is an orchestration layer

It doesn't reinvent every AWS capability — it **orchestrates existing ones**:

```
                 AWS CONTROL TOWER
                        |
        ┌───────────────┼────────────────┐
        ↓               ↓                ↓
   Organizations   Service Catalog   IAM Identity Center
        |
        ↓
   Other AWS services
        |
        ↓
   Governed environment
```

Understanding Control Tower means understanding **what it manages/orchestrates**, not just its console.

---

## 27. Custom requirements

Real companies rarely want only the default config:

```
AWS default governance
        + Company security standards
        + Company networking
        + Company compliance
        + Company tagging
        + Company deployment standards
```

Control Tower can be extended/customized — but **only through supported methods**, never by manually editing managed resources.

---

## 28. Important operational rule

**Do not casually modify Control Tower–managed resources.**

```
Control Tower creates/controls resource
              ↓
Administrator manually changes/deletes it
              ↓
Control Tower configuration may drift
              ↓
Landing zone can become unhealthy/unknown
```

Always use supported Control Tower mechanisms.

---

## 29. Lifecycle — simple view

```
              PLAN
                ↓
       Set up Control Tower
                ↓
         Create Landing Zone
                ↓
      Configure governance
                ↓
       Provision/enroll accounts
                ↓
        Apply/manage controls
                ↓
       Monitor compliance
                ↓
       Detect drift/problems
                ↓
        Update/maintain
                ↓
          Scale environment
```

---

## 30. What to remember

| Term | Meaning |
|---|---|
| **Control Tower** | Manages governance for a multi-account AWS environment |
| **Landing Zone** | The governed, well-architected multi-account environment |
| **Account Factory** | Standardized account provisioning |
| **Controls** | Rules for ongoing governance |
| **Dashboard** | Central visibility into governance and compliance |
| **Drift** | Environment differs from expected managed configuration |
| **Cost** | No extra Control Tower charge; underlying services cost money |

---

## 31. Interview-ready answer

**Q: What is AWS Control Tower?**

**A:** AWS Control Tower is an AWS service that helps organizations set up and govern a secure, multi-account AWS environment using AWS best practices. It provides a governed landing zone, controls for ongoing governance, Account Factory for standardized account provisioning, and a dashboard for centralized visibility and compliance monitoring.

---

## 32. Short answer for a meeting

> "We use AWS Control Tower to establish a standardized multi-account AWS environment and maintain centralized governance. It helps with landing-zone setup, account provisioning, preventive/detective/proactive controls, compliance visibility, and ongoing governance across accounts."

---

## 33. Easy memory trick

```
CONTROL TOWER
      |
      ↓
"Control my AWS environment"
      |
      ├── LANDING ZONE       → Foundation
      ├── ACCOUNT FACTORY    → New accounts
      ├── CONTROLS           → Governance rules
      └── DASHBOARD          → Visibility
```

**The complete mental model:**

```
             🏢 ENTERPRISE AWS
                    |
                    ↓
             AWS CONTROL TOWER
                    |
       ┌────────────┼────────────┐
       ↓            ↓            ↓
   Foundation   Governance    Visibility
       ↓            ↓            ↓
 Landing Zone    Controls     Dashboard
       |
       ↓
 Account Factory
       |
       ↓
 New / Enrolled Accounts
       |
       ↓
 Ongoing Governance
```

---

## 34. Final takeaway

> AWS Control Tower is a governance service for multi-account AWS environments. It establishes a standardized landing zone, helps provision accounts consistently, applies governance controls, and provides centralized visibility into compliance and configuration.

```
Multiple AWS Accounts
        ↓
Need Standardization
        ↓
Need Governance
        ↓
Need Compliance Visibility
        ↓
        AWS CONTROL TOWER
```

---

