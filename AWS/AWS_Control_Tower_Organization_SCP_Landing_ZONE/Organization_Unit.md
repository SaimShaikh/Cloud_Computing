
# AWS Organizational Unit (OU) — Easy Guide

> Learn OUs from zero, in simple words, without losing the important technical details.
> Scope: focuses only on AWS Organizational Units (OUs). Other AWS concepts are mentioned only when needed to explain how an OU works.

---

## 1. What is an Organizational Unit (OU)?

**OU = Organizational Unit.**

An OU is a **logical group of AWS accounts inside an AWS Organization**.

> **OU = putting similar AWS accounts into one logical group.**

```
AWS Organization
│
├── Development OU
│   ├── Dev-Account-1
│   ├── Dev-Account-2
│   └── Dev-Account-3
│
├── Testing OU
│   ├── Test-Account-1
│   └── Test-Account-2
│
└── Production OU
    ├── Prod-Account-1
    └── Prod-Account-2
```

- Development OU contains development accounts.
- Testing OU contains testing accounts.
- Production OU contains production accounts.

---

## 2. Why do we need OUs?

Imagine a company has 100 AWS accounts. Without OUs:

```
Account 1
Account 2
Account 3
Account 4
...
Account 100
```

Managing them individually becomes difficult. With OUs:

```
Development OU  → 10 Dev accounts
Testing OU      → 10 Test accounts
Production OU   → 30 Production accounts
Security OU     → Security accounts
Sandbox OU      → 50 Sandbox accounts
```

Now the environment is easier to organize and govern.

---

## 3. When should we use an OU?

Use OUs when you need to:
- Organize multiple AWS accounts
- Separate environments
- Separate business units
- Separate security boundaries
- Apply common policies to groups of accounts
- Apply governance consistently
- Manage a large AWS organization

**When it's not needed:** for a single AWS account, an OU has no practical organizational benefit.

---

## 4. Easy real-world example

A company's departments have different responsibilities and rules:

```
ABC Company
│
├── HR
├── Finance
├── Development
├── Testing
└── Production
```

AWS can use a similar logical structure:

```
AWS Organization
│
├── Development OU
├── Testing OU
├── Production OU
└── Security OU
```

**Important idea:** OU groups AWS accounts based on organizational needs.

---

## 5. OU is NOT an AWS Account

This is important.

**AWS Account** — the actual AWS environment where resources run:
```
AWS Account
├── EC2
├── S3
├── RDS
├── VPC
└── Lambda
```

**OU** — a container/group *for* AWS accounts:
```
Development OU
├── Dev Account 1
├── Dev Account 2
└── Dev Account 3
```

So: `OU ≠ AWS Account`

> **Account = individual unit**
> **OU = group of accounts**

---

## 6. OU vs IAM Group — similar idea, different purpose

**IAM Group** — groups **IAM users**:
```
Developers Group
├── User A
├── User B
└── User C
```
Common permissions can then be attached to the group.

**OU** — groups **AWS accounts**:
```
Development OU
├── Dev Account 1
├── Dev Account 2
└── Dev Account 3
```
Policies/controls can then be targeted at the OU.

| IAM Group | AWS OU |
|---|---|
| Groups users | Groups AWS accounts |
| Used for identity permissions | Used for organization/governance |
| Contains IAM users | Contains AWS accounts and possibly child OUs |
| Part of IAM | Part of AWS Organizations |

**Memory trick:**
```
IAM Group → User → Group → Permissions
OU        → Account → OU → Policies / Governance
```

The concepts are similar as a grouping idea, but the AWS mechanisms and purposes are different.

---

## 7. How does an OU work?

```
Create AWS Organization
        ↓
Create OU
        ↓
Create or move AWS accounts
        ↓
Put accounts into the appropriate OU
        ↓
Apply organizational policies where required
        ↓
Accounts inherit the applicable organizational restrictions
```

Example:
```
AWS Organization
        ↓
Development OU
        ↓
Dev-1, Dev-2, Dev-3
```

---

## 8. Can an OU contain users?

**No.**

An OU is designed to contain:
- AWS accounts
- Child OUs

It is not an IAM user group.

**Correct:**
```
Development OU
├── Dev Account 1
├── Dev Account 2
└── Dev Account 3
```

**Incorrect:**
```
Development OU
├── Rahul ❌
├── Amit ❌
└── John ❌
```

Users are handled by identity/access services, not OUs.

---

## 9. Can an OU contain another OU?

**Yes.** OUs can be nested.

```
Root
│
└── Production OU
    │
    ├── Critical Production OU
    │   ├── Prod Account 1
    │   └── Prod Account 2
    │
    └── Standard Production OU
        ├── Prod Account 3
        └── Prod Account 4
```

This is called a **nested OU structure**.

---

## 10. Why would we nest OUs?

Suppose Production needs sub-groups with different governance:

```
Production
├── Critical
├── Standard
└── Regulated
```

Represented logically:
```
Production OU
│
├── Critical OU
├── Standard OU
└── Regulated OU
```

This allows the organization to build a hierarchy.

---

## 11. OU hierarchy

A simplified AWS Organizations hierarchy:

```
Root
│
├── Development OU
│   ├── Dev Account 1
│   └── Dev Account 2
│
├── Testing OU
│   ├── Test Account 1
│   └── Test Account 2
│
└── Production OU
    ├── Prod Account 1
    └── Prod Account 2
```

With nested OUs:

```
Root
│
└── Production OU
    │
    ├── Critical OU
    │   ├── Prod-1
    │   └── Prod-2
    │
    └── Standard OU
        ├── Prod-3
        └── Prod-4
```

---

## 12. The most important reason for OUs

> **OUs let you manage groups of accounts together instead of managing every account individually.**

**Without OU:**
```
Policy → Account 1
Policy → Account 2
Policy → Account 3
Policy → Account 4
...
```

**With OU:**
```
Policy
  ↓
Development OU
  ↓
Dev 1, Dev 2, Dev 3, Dev 4
```

This makes organizational governance easier.

---

## 13. Policies and OUs

An OU becomes especially useful when organizational policies need to apply to a group of accounts:

```
Development OU
      |
      ↓
Organizational policy
      |
 ┌────┼────┐
 ↓    ↓    ↓
Dev1 Dev2 Dev3
```

The accounts in the OU are affected according to the policy's scope and inheritance rules.

---

## 14. SCP and OU

A common example is a **Service Control Policy (SCP)**.

Suppose the organization wants to restrict a particular AWS action for all Development accounts:

```
SCP
 ↓
Development OU
 ↓
Dev Account 1
Dev Account 2
Dev Account 3
```

Instead of attaching the restriction separately to every account, it can be applied at the OU level.

> ⚠️ An SCP does **not** grant permissions. Think of it as the **maximum permission boundary** at the organization level — an IAM policy is still needed to actually grant a user/role permission.

---

## 15. Policy inheritance

If a policy is attached at a higher level, it can affect child OUs/accounts according to AWS Organizations inheritance rules.

```
Root
│
└── Production OU
    │
    ├── Prod-1
    └── Prod-2
```

If a policy is attached to Production OU:
```
Policy
  ↓
Production OU
  ↓
Prod-1, Prod-2
```
Child accounts are affected.

With nested OUs:
```
Root
 ↓
Production OU
 ↓
Critical Production OU
 ↓
Prod-1
```

A policy at the higher level can flow down to the child hierarchy, subject to the specific policy type and AWS Organizations rules.

---

## 16. Important: OUs do NOT automatically give permissions

A very common mistake:

❌ **Incorrect:** "If I put an account in the Development OU, developers automatically get access."

OU is about **organization and governance**, not human login permissions.

✅ **Correct:**
```
OU              → Account organization / organizational policies
Identity system → User access / permissions
```

These are separate concerns.

---

## 17. How should we design OUs?

Don't create OUs randomly. First ask:

> **What differences in governance do I need to represent?**

For example:
- Do Production accounts need stricter policies?
- Do Sandbox accounts need different restrictions?
- Do regulated workloads need different controls?
- Do different business units need different governance?

Then design the OU hierarchy around those requirements.

---

## 18. Common OU design — Environment based

```
Root
│
├── Development OU
├── Testing OU
├── Staging OU
└── Production OU
```

**When useful?** When the biggest difference between accounts is their environment.

---

## 19. Common OU design — Security based

```
Root
│
├── Security OU
├── Infrastructure OU
├── Workloads OU
└── Sandbox OU
```

Useful when governance requirements are based on account purpose.

---

## 20. Common OU design — Business based

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

This can work when business-unit governance is the primary requirement.

---

## 21. Which OU design is best?

There is **no single OU structure that is best for every company**. The right design depends on:
- Governance requirements
- Security requirements
- Compliance requirements
- Account lifecycle
- Business structure
- Networking needs
- Policy differences

**A good rule:** create an OU when accounts need to be managed differently from other accounts. Don't create dozens of OUs just because you can.

---

## 22. Bad OU design

```
Root
├── India OU
├── Pune OU
├── Mumbai OU
├── Bangalore OU
├── Team-A OU
├── Team-B OU
├── Project-1 OU
├── Project-2 OU
...
```

If these groups don't require different governance, the hierarchy becomes unnecessarily complicated.

---

## 23. Good OU design

Instead, organize around meaningful governance boundaries:

```
Root
│
├── Security OU
│
├── Sandbox OU
│
├── Non-Production OU
│   ├── Development
│   └── Testing
│
└── Production OU
    ├── Critical
    └── Standard
```

This is only an example — real organizations should design based on actual requirements.

---

## 24. Moving an account between OUs

An AWS account can be moved between OUs when organizational requirements change, subject to AWS Organizations rules and permissions.

```
Before:
Development OU
└── Payment Account

After:
Production OU
└── Payment Account
```

When the account moves, it becomes subject to the policies associated with the destination OU, and no longer subject to policies that were only associated with the previous OU.

This is why OU placement matters.

---

## 25. Scenario 1 — Development accounts

```
Development OU
├── Dev-1
├── Dev-2
├── Dev-3
└── Dev-4
```

Now common organizational restrictions can be applied to the Development OU.

---

## 26. Scenario 2 — Production accounts

Production needs stricter governance:

```
Production OU
├── Prod-1
├── Prod-2
└── Prod-3
```

```
Production OU
      ↓
Stricter governance
      ↓
Prod accounts
```

---

## 27. Scenario 3 — Critical production

Suppose two production applications are extremely sensitive:

```
Production OU
│
├── Standard Production OU
│   ├── Prod-1
│   └── Prod-2
│
└── Critical Production OU
    ├── Critical-Prod-1
    └── Critical-Prod-2
```

Different governance can be applied at different levels.

---

## 28. Scenario 4 — Sandbox

```
Sandbox OU
├── Sandbox-1
├── Sandbox-2
└── Sandbox-3
```

The organization may choose different restrictions for sandbox accounts compared with Production.

---

## 29. Advantages

1. **Organization** — makes a large number of accounts easier to understand
2. **Centralized policy targeting** — policies can be targeted at groups of accounts
3. **Easier governance** — similar accounts can share organizational rules
4. **Scalability** — useful as the number of accounts grows
5. **Clear separation** — environment/business/security boundaries become visible
6. **Easier account management** — accounts can be moved between groups as requirements change

---

## 30. Limitations / disadvantages

1. **Hierarchy complexity** — too many OUs can make the organization hard to understand
2. **Poor design creates management problems** — OUs without a governance reason become messy
3. **Not an access-control system for users** — doesn't replace IAM or identity management
4. **Doesn't secure workloads by itself** — putting an account in an OU doesn't automatically secure its applications
5. **Policy inheritance can get complicated** — nested OUs with multiple policies need careful understanding

---

## 31. Cost / Billing

**Does creating an OU cost money?** No separate charge is normally associated with simply creating an OU — an OU is an organizational construct.

However, the policies, services, and resources used *in* the accounts can have costs:

```
OU → Accounts → AWS resources → AWS service charges
```

The OU itself is not a workload resource like EC2 or NAT Gateway.

---

## 32. OU and AWS Control Tower

```
AWS Control Tower
        ↓
Governed multi-account environment
        ↓
OUs
        ↓
Accounts
```

Control Tower can use and manage organizational structure as part of its governance model.

> **Control Tower uses the multi-account organizational structure; OUs provide the logical grouping of accounts.**

---

## 33. OU architecture

```
                     AWS ORGANIZATION
                            |
                           Root
                            |
        ┌───────────────────┼───────────────────┐
        ↓                   ↓                   ↓
  Development OU       Non-Production OU    Production OU
        |                   |                   |
    ┌───┼───┐           ┌───┴───┐          ┌───┴───┐
    ↓   ↓   ↓           ↓       ↓          ↓       ↓
  Dev1 Dev2 Dev3       Test1   Test2      Prod1   Prod2
```

---

## 34. Nested OU architecture

```
                         Root
                          |
                    Production OU
                          |
              ┌───────────┴───────────┐
              ↓                       ↓
      Standard Prod OU        Critical Prod OU
              |                       |
          ┌───┴───┐               ┌───┴───┐
          ↓       ↓               ↓       ↓
        Prod1   Prod2          Prod3    Prod4
```

Policies can be applied at appropriate levels according to the governance design.

---

## 35. Rules to remember

| # | Rule |
|---|---|
| 1 | OU contains AWS accounts, not users |
| 2 | OU is used to organize accounts |
| 3 | OUs can contain child OUs |
| 4 | Policies can be targeted at OUs |
| 5 | OU does not directly grant human permissions |
| 6 | Good OUs represent meaningful governance boundaries |
| 7 | Don't create OUs just for the sake of creating them |

---

## 36. Interview-ready answer

**Q: What is an AWS Organizational Unit?**

**A:** An Organizational Unit, or OU, is a logical container within AWS Organizations used to group AWS accounts based on organizational, security, or governance requirements. OUs allow organizations to structure their accounts hierarchically and apply organizational policies to groups of accounts rather than managing each account individually.

---

## 37. Short answer for a meeting

> "An OU is basically a logical group of accounts. For example, we can put all Development accounts into a Development OU. If those accounts share a common governance requirement, it's easier to manage that requirement at the OU level. OUs can also be nested, so we can build a hierarchy based on our organization's requirements."

---

## 38. Easy memory trick

```
ACCOUNT → Individual AWS environment
OU      → Group of AWS accounts
POLICY  → Rules applied to the organizational structure
USER    → Identity / access management
```

**Most important line:** OU is the AWS Organizations mechanism for logically grouping accounts — mainly to make organizational management and policy application easier.

---

## 39. Final mental model

```
                         AWS ORGANIZATION
                                |
                               Root
                                |
             ┌──────────────────┼──────────────────┐
             ↓                  ↓                  ↓
        Development OU      Security OU       Production OU
             |                  |                  |
        ┌────┼────┐             ↓             ┌────┼────┐
        ↓    ↓    ↓        Security Acct      ↓    ↓    ↓
      Dev1  Dev2  Dev3                       Prod1 Prod2 Prod3
```

Think of it like this:
```
AWS Account = one house 🏠
OU          = group/society of similar houses 🏘️
Policy      = rules for that group 📜
```

---

## Final takeaway

> An OU is not a new AWS account and not a user group. It is a logical container for AWS accounts (and child OUs). Its main value is giving the organization a clean hierarchy and a practical place to apply organizational governance to groups of accounts.
