# AWS Organizations — Easy Guide

> Learn AWS Organizations from zero, in simple words, without losing the important technical details.
> Scope: focuses on AWS Organizations itself. Related concepts (OU, Control Tower, SCP) are mentioned only where needed to explain how Organizations works.

---

## 1. What is AWS Organizations?

**AWS Organizations** is an AWS service that lets you **centrally manage and govern multiple AWS accounts** as a single organization.

> **AWS Organizations = the service that groups many AWS accounts together under one root, so they can be managed, billed, and governed centrally.**

```
AWS Organization
│
├── Management Account
│
├── Development OU
│   ├── Dev-Account-1
│   └── Dev-Account-2
│
├── Production OU
│   ├── Prod-Account-1
│   └── Prod-Account-2
│
└── Security OU
    └── Security-Account
```

**One-line definition:**
AWS Organizations = central account management + hierarchy (OUs) + policies (SCPs, etc.) + consolidated billing for many AWS accounts.

---

## 2. Why do we need it?

Imagine a company has 50 separate AWS accounts, each signed up independently:

```
Account 1 → own billing, own root user, own policies
Account 2 → own billing, own root user, own policies
Account 3 → own billing, own root user, own policies
...
```

This creates:
- Separate bills for every account
- No central way to apply security rules
- No structure/hierarchy
- Difficult governance
- Difficult auditing
- Accounts managed in isolation

AWS Organizations solves this by giving you **one place** to create, organize, bill, and govern all these accounts together.

---

## 3. When should you use it?

Useful when:
- You have (or expect to have) more than one AWS account
- You want a single consolidated bill
- You need to group accounts logically (OUs)
- You need to apply organization-wide policies
- You need centralized account creation
- You're planning to adopt Control Tower or a Landing Zone (both are built on top of Organizations)

**When it may not be necessary:** if you only ever need a single AWS account with no plans to expand, Organizations adds little value.

---

## 4. The big picture

```
                    AWS ORGANIZATIONS
                            |
        ┌───────────────────┼───────────────────┐
        ↓                   ↓                   ↓
   Account structure     Policies          Consolidated billing
        |                   |
        ↓                   ↓
    Root → OUs → Accounts   SCPs / other policy types
```

**AWS Organizations** is the foundational service.
**OUs**, **Control Tower**, and a **Landing Zone** are all built using Organizations underneath.

---

## 5. Key building blocks

| Term | Simple meaning |
|---|---|
| Root | The top-level container of the organization |
| Management account | The account that creates and owns the organization |
| Member account | Any other account that belongs to the organization |
| OU (Organizational Unit) | A logical group of accounts inside the organization |
| Policy | A rule attached to the root, an OU, or an account |
| Consolidated billing | One combined bill for every account in the organization |

---

## 6. Root

The **Root** is the top of the organization's hierarchy — every OU and account exists somewhere underneath it.

```
Root
│
├── Development OU
├── Testing OU
├── Production OU
└── Security OU
```

Think of Root as the "trunk" of the organizational tree.

---

## 7. Management account

The **management account** is the account used to **create** the AWS Organization.

It's special:
- It owns/administers the organization
- It's where consolidated billing is viewed
- It can create new accounts and invite existing ones
- It can attach organization-wide policies

```
Management Account
        ↓
Creates/Owns
        ↓
AWS Organization
```

> ⚠️ Best practice: don't run everyday workloads (EC2, S3, apps, etc.) in the management account. Treat it as an administrative account, not a workload account.

---

## 8. Member accounts

**Member accounts** are all the other accounts that join the organization — either newly created through Organizations, or invited in from existing standalone accounts.

```
AWS Organization
│
├── Management Account
│
└── Member Accounts
    ├── Dev-Account
    ├── Test-Account
    ├── Prod-Account
    └── Security-Account
```

Member accounts still function as normal AWS accounts (you deploy EC2, S3, RDS, etc. inside them) — but they're now subject to the organization's structure and policies.

---

## 9. How does a member account join?

Two common ways:

**A) Create a new account directly through Organizations:**
```
Management Account
        ↓
Create new account (via Organizations)
        ↓
New account automatically joins the organization
```

**B) Invite an existing standalone account:**
```
Existing AWS account
        ↓
Invitation sent via Organizations
        ↓
Account owner accepts
        ↓
Account joins the organization
```

---

## 10. Organizational Units (OUs) — quick recap

An OU is a **logical group of accounts** inside the organization, used to apply policies and structure consistently.

```
Root
│
├── Development OU
│   ├── Dev-1
│   └── Dev-2
│
└── Production OU
    ├── Prod-1
    └── Prod-2
```

> OUs are a feature *of* AWS Organizations — you can't have OUs without Organizations underneath them.

---

## 11. Policies in AWS Organizations

AWS Organizations supports several **policy types**, each governing a different aspect of member accounts. The most common is the **Service Control Policy (SCP)**.

| Policy type | Simple meaning |
|---|---|
| Service Control Policy (SCP) | Sets the maximum allowed AWS actions for accounts/OUs |
| Tag policy | Standardizes how resources are tagged across accounts |
| Backup policy | Standardizes backup plans across accounts |
| AI services opt-out policy | Controls whether AI services can use account data for improvement |

```
Policy
  ↓
Attached to Root / OU / Account
  ↓
Applies to everything underneath it
```

---

## 12. Service Control Policies (SCPs) — the most important one

An **SCP** defines the **maximum permissions** available to accounts in an OU (or the whole organization).

> ⚠️ An SCP does **not grant** permissions by itself — it only sets a ceiling. An IAM policy inside the account is still required to actually grant a user or role permission.

```
SCP (maximum boundary)
        ↓
IAM policy (actual grant)
        ↓
Effective permission = smaller of the two
```

**Example:** An SCP on the Sandbox OU could block accounts from launching very large/expensive EC2 instance types, regardless of what IAM permissions exist inside those accounts.

---

## 13. Policy inheritance

Policies attached higher in the hierarchy flow down to everything beneath them.

```
Root
│
└── Production OU
    │
    ├── Critical Prod OU
    │   └── Prod-1
    │
    └── Standard Prod OU
        └── Prod-2
```

A policy attached at **Production OU** affects both **Critical Prod OU** and **Standard Prod OU**, and therefore Prod-1 and Prod-2 as well — subject to how each specific policy type combines with policies at other levels.

---

## 14. Consolidated billing

One of the most immediately practical benefits of AWS Organizations.

```
Dev Account    ─┐
Test Account   ─┤
Prod Account   ─┼──→ Consolidated Bill (on Management Account)
Security Acct  ─┘
```

**Benefits:**
- One combined invoice instead of many
- Easier cost tracking and allocation across accounts
- Access to volume pricing/discounts that combine usage across accounts, where applicable
- Each member account can still see its own usage/costs

---

## 15. AWS Organizations vs OU vs Control Tower vs Landing Zone

This is where people often get confused — here's the simple distinction:

| Concept | What it actually is |
|---|---|
| **AWS Organizations** | The underlying service that creates accounts, structure (root/OUs), and policies |
| **OU** | A grouping feature *inside* Organizations |
| **Landing Zone** | The overall governed *environment/foundation* an organization builds (using Organizations, and often Control Tower) |
| **Control Tower** | A managed AWS service that sets up and governs a Landing Zone, using Organizations underneath it |

```
AWS Organizations (foundation)
        ↓
    OUs (structure inside it)
        ↓
Control Tower (managed governance layer, optional)
        ↓
Landing Zone (the resulting governed environment)
```

> **Bottom line:** you cannot have OUs, Control Tower, or a Landing Zone without AWS Organizations underneath — Organizations is the base layer everything else builds on.

---

## 16. A typical setup flow

```
Create AWS account
        ↓
Enable AWS Organizations
        ↓
Management account created
        ↓
Create OUs (Dev, Test, Prod, Security, Sandbox, etc.)
        ↓
Create or invite member accounts
        ↓
Place accounts into the right OU
        ↓
Attach policies (SCPs, tag policies, etc.) where needed
        ↓
Review consolidated billing
        ↓
Organization is operational
```

---

## 17. Scenario: startup growing into multiple accounts

A startup begins with one AWS account. As it grows:

```
1 account
    ↓
"We need separate Dev and Prod"
    ↓
Enable AWS Organizations
    ↓
Create Dev OU, Prod OU
    ↓
Create Dev account, keep existing account as Prod
    ↓
Apply basic SCPs (e.g., restrict risky regions/actions in Prod)
    ↓
One consolidated bill for both accounts
```

---

## 18. Scenario: enterprise with many business units

A large enterprise has Finance, Retail, and HR business units, each needing Dev/Prod separation:

```
Root
│
├── Finance OU
│   ├── Finance-Dev
│   └── Finance-Prod
│
├── Retail OU
│   ├── Retail-Dev
│   └── Retail-Prod
│
└── HR OU
    ├── HR-Dev
    └── HR-Prod
```

Each business unit's OU can carry its own governance requirements, while the whole organization still shares one consolidated bill and one root-level policy baseline.

---

## 19. Scenario: security restriction across the whole org

Company wants: *"No account, anywhere in the organization, should be able to disable CloudTrail."*

```
SCP: "Deny disabling CloudTrail"
        ↓
Attached at Root
        ↓
Applies to every OU and account in the organization
```

Applying it once at Root is far simpler than configuring it account-by-account.

---

## 20. Advantages

1. **Centralized account management** — create, organize, and manage many accounts from one place
2. **Consolidated billing** — one bill, easier cost visibility
3. **Centralized governance** — policies applied at root/OU level instead of per account
4. **Structured hierarchy** — OUs bring order to large numbers of accounts
5. **Foundation for other services** — Control Tower, SSO/IAM Identity Center, and Landing Zones all build on it
6. **Security isolation** — separate accounts (e.g., Security, Log Archive) limit blast radius
7. **Scales well** — works whether you have 5 accounts or 500

---

## 21. Disadvantages / limitations

1. **Added complexity** — hierarchy, policies, and inheritance need to be understood
2. **Management account is sensitive** — mismanaging it can affect the whole organization
3. **SCPs are not permissions grants** — teams often misunderstand this and expect SCPs alone to "give" access
4. **Migration effort** — bringing many pre-existing standalone accounts into an organization takes planning
5. **Policy inheritance can be confusing** — especially with deeply nested OUs and multiple policy types
6. **Not a complete governance solution by itself** — still often paired with Control Tower, SCPs, logging, and monitoring for full governance

---

## 22. Billing / Cost

**Most important fact:** AWS Organizations itself has **no additional charge** for creating accounts, OUs, or basic policies.

You pay for:
- The AWS resources actually used inside each member account (EC2, S3, RDS, etc.)
- Any AWS services enabled for governance (CloudTrail, Config, etc. — same story as with Control Tower)

```
AWS Organizations → $0 service charge
Accounts inside it → Normal AWS usage charges
```

Consolidated billing doesn't add cost — it just **combines** the existing costs from every member account into one invoice.

---

## 23. What AWS Organizations does NOT mean

❌ **It does not mean** every account automatically gets secured — SCPs set boundaries, but IAM, logging, and monitoring still need to be configured.

❌ **It does not mean** accounts share resources — each member account still has its own isolated VPCs, S3 buckets, EC2 instances, etc.

❌ **It does not mean** it replaces Control Tower — Organizations is the foundation; Control Tower is an optional managed layer on top for easier governance.

---

## 24. Important operational rule

Don't casually delete or restructure OUs/accounts without understanding policy impact.

```
Account moved to a different OU
              ↓
It gains the new OU's policies
              ↓
It loses policies that were only attached to the old OU
              ↓
Unexpected access changes can occur if not reviewed carefully
```

Always review which policies apply before moving accounts between OUs.

---

## 25. Architecture — simple view

```
                    AWS ORGANIZATION
                            |
                           Root
                            |
        ┌───────────────────┼───────────────────┐
        ↓                   ↓                   ↓
  Development OU       Security OU         Production OU
        |                   |                   |
    ┌───┼───┐                ↓              ┌────┼────┐
    ↓   ↓   ↓           Security Acct       ↓    ↓    ↓
  Dev1 Dev2 Dev3                          Prod1 Prod2 Prod3

        Policies (SCPs, tag policies, etc.) attached at
        Root / OU / Account level, flowing downward
```

---

## 26. What to remember

| Term | Meaning |
|---|---|
| **AWS Organizations** | The service that centrally manages multiple AWS accounts |
| **Root** | Top of the account hierarchy |
| **Management account** | The account that owns/administers the organization |
| **Member account** | Any other account inside the organization |
| **OU** | Logical grouping of accounts inside Organizations |
| **SCP** | Sets the maximum allowed permissions for accounts/OUs |
| **Consolidated billing** | One combined invoice for all accounts |
| **Cost** | No extra charge for Organizations itself; usual AWS usage costs apply |

---

## 27. Interview-ready answer

**Q: What is AWS Organizations?**

**A:** AWS Organizations is an AWS service that lets you centrally create, manage, and govern multiple AWS accounts. It provides a hierarchical structure using a root and Organizational Units (OUs), supports policies such as Service Control Policies to govern what actions are allowed across accounts, and provides consolidated billing so all accounts in the organization are billed together.

---

## 28. Short answer for a meeting

> "We use AWS Organizations to manage all our AWS accounts under one structure. It lets us group accounts into OUs, apply governance policies like SCPs at the organization or OU level instead of per account, and get one consolidated bill across everything. It's also the foundation that services like Control Tower and our Landing Zone are built on."

---

## 29. Easy memory trick

```
AWS ORGANIZATIONS
      |
      ↓
"The foundation for managing many AWS accounts"
      |
      ├── ROOT              → Top of the hierarchy
      ├── OUs                → Logical account groups
      ├── ACCOUNTS           → Management + Member accounts
      ├── POLICIES (SCPs...) → Governance rules
      └── CONSOLIDATED BILL  → One invoice for everything
```

---

## 30. Final mental model

```
                    AWS ORGANIZATIONS
                            |
                           Root
                            |
        ┌───────────────────┼───────────────────┐
        ↓                   ↓                   ↓
   Account Structure     Policies          Consolidated Billing
        |                   |                       |
     Root → OUs         SCPs, Tag policies,     One invoice for
     → Accounts         Backup policies, etc.   all accounts
        |
        ↓
  Optional layer: Control Tower
        |
        ↓
  Result: a governed Landing Zone
```

---

## Final takeaway

> AWS Organizations is the **foundational service** for managing multiple AWS accounts. It gives you a hierarchy (root and OUs), a way to apply governance policies like SCPs across many accounts at once, and consolidated billing — and it's the base layer that features like Control Tower, Landing Zones, and cross-account governance are all built on top of.
