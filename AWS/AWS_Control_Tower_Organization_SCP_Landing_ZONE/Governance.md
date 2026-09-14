

# AWS Service Control Policies (SCPs) — Detailed Easy & Professional Guide

> **Scope:** This guide focuses on AWS Service Control Policies (SCPs). Other AWS concepts are included only when needed to understand SCPs.

## 1. What is an SCP?

**SCP = Service Control Policy**

An SCP is an AWS Organizations policy that defines the **maximum permissions** available to IAM users and IAM roles in member AWS accounts.

Simple meaning:

> **SCP = organization-level guardrail that limits what IAM users and roles can do.**

Most important rule:

> **SCP does NOT grant permissions. It only sets a permission boundary at the organization level.**

---

## 2. The easiest way to understand SCP

Think about a company rule:

> "Employees may enter the building, but nobody is allowed to enter the server room."

In AWS:

```text
IAM Policy
    ↓
May grant permission

SCP
    ↓
Can restrict that permission

Effective access
    ↓
Only what the applicable policies allow
```

Memory:

```text
IAM → "You can do this."
SCP → "But the organization does not permit this."
```

---

## 3. SCP does NOT give access

Suppose:

```text
SCP:
Allow EC2

IAM:
No EC2 permissions
```

Can the user launch EC2?

**No.**

The SCP does not grant the permission.

Correct model:

```text
SCP
  ↓
Maximum allowed permissions

IAM Policy
  ↓
Permissions actually granted

Effective permission
  ↓
What both sides permit
```

---

## 4. SCP vs IAM Policy

| Feature | IAM Policy | SCP |
|---|---|---|
| Main purpose | Grant/control permissions | Set organization-level guardrails |
| Grants permission? | Yes, depending on policy type | No |
| Scope | Users, roles, resources, etc. | Organization, root, OU, account |
| Main use | Give access | Limit maximum access |
| Managed by | IAM | AWS Organizations |
| Example | Allow S3 read | Deny S3 delete |

Easy memory:

```text
IAM → grants permissions
SCP → sets maximum permissions
```

---

## 5. Why do we need SCPs?

Imagine an organization has:

```text
AWS Organization
│
├── Development Account
├── Testing Account
├── Production Account
├── Security Account
└── Finance Account
```

You may want organization-wide guardrails such as:

- do not use prohibited AWS Regions
- do not disable important logging
- do not perform specific dangerous actions
- do not use prohibited services
- protect critical production accounts
- prevent certain organization-management actions

Instead of relying only on each account administrator, SCPs provide centralized organizational restrictions.

---

## 6. Where can an SCP be attached?

An SCP can be attached to:

```text
Root
OU
AWS Account
```

Example:

```text
Organization Root
       |
       ├── Development OU
       |      ├── Dev-1
       |      └── Dev-2
       |
       └── Production OU
              ├── Prod-1
              └── Prod-2
```

You can choose the attachment level based on the required scope.

---

## 7. SCP hierarchy

Example:

```text
Root
│
└── Production OU
      │
      ├── Critical Production OU
      │     ├── Prod-1
      │     └── Prod-2
      │
      └── Standard Production OU
            ├── Prod-3
            └── Prod-4
```

An SCP attached higher in the hierarchy can affect lower-level accounts according to AWS Organizations policy evaluation and inheritance rules.

Conceptually:

```text
Production SCP
      ↓
Child OUs
      ↓
Member accounts
```

---

## 8. SCP inheritance

A member account can be affected by applicable SCPs attached at different levels.

For example:

```text
Root
 ↓
Production OU
 ↓
Prod Account
```

The account can be subject to applicable restrictions from:

```text
Root SCP
   +
OU SCP
   +
Account SCP
```

Think:

```text
Higher-level guardrails
        +
More-specific guardrails
        ↓
Effective organizational restrictions
```

---

## 9. SCP evaluation — core idea

A request is successful only when the applicable authorization policies and organizational controls permit it.

Simplified model:

```text
IAM permissions
       ∩
Applicable SCP permissions/guardrails
       ∩
Other applicable controls
       =
Effective permissions
```

The exact AWS policy evaluation process has additional details, but this model is the most useful way to remember SCPs.

---

## 10. Explicit Deny

Suppose IAM says:

```text
Allow:
ec2:RunInstances
```

But SCP says:

```text
Deny:
ec2:RunInstances
```

Result:

```text
DENIED
```

Memory:

```text
IAM → Allow
SCP → Explicit Deny
        ↓
     DENIED
```

An explicit deny overrides an allow.

---

## 11. Deny-list and allow-list strategies

There are two common SCP strategies.

### Strategy A — Deny list

Concept:

```text
Allow normal AWS access
       +
Deny specific dangerous actions
```

Example:

```text
Deny:
- Disable CloudTrail
- Leave organization
- Use prohibited Region
- Perform selected destructive actions
```

This is often easier to introduce gradually.

### Strategy B — Allow list

Concept:

```text
Allow only approved services/actions
```

Example:

```text
Allowed:
EC2
S3
CloudWatch
SSM

Other services/actions:
not permitted by the SCP structure
```

This is much stricter and requires careful planning and testing.

---

## 12. FullAWSAccess policy

When SCPs are enabled, AWS Organizations uses the default `FullAWSAccess` policy in the standard SCP model.

Conceptually:

```text
FullAWSAccess
      ↓
Does not restrict AWS services/actions at the SCP layer
```

Important:

```text
FullAWSAccess
≠
Everyone gets AdministratorAccess
```

It does **not** grant IAM permissions.

IAM policies are still required to grant actual access.

If you remove `FullAWSAccess`, you need an appropriate replacement allow policy structure; otherwise actions can fail for member accounts.

---

## 13. Management account exception

SCPs do **not** restrict users or roles in the AWS Organizations **management account**.

Conceptually:

```text
AWS Organization
│
├── Management Account
│      └── SCPs do not restrict its users/roles
│
└── Member Accounts
       ├── Account A ← SCPs apply
       ├── Account B ← SCPs apply
       └── Account C ← SCPs apply
```

Do not confuse the management account with member accounts.

---

## 14. SCPs and member-account root users

SCPs generally apply to principals in member accounts, including the member account root user, subject to AWS documented exceptions.

This is different from the management account.

---

## 15. Service-linked roles

SCPs do not restrict service-linked roles.

Service-linked roles are special IAM roles used by AWS services to integrate with other AWS services.

This is an important SCP exception to remember.

---

## 16. SCP does not directly replace resource policies

SCPs primarily control the maximum permissions available to IAM users and roles in member accounts.

For example:

```text
SCP
 ↓
Limits organizational permissions
```

while:

```text
S3 Bucket Policy
 ↓
Controls access to an S3 resource
```

AWS also has **Resource Control Policies (RCPs)** for resource-focused organizational controls.

Memory:

```text
SCP → IAM principals
RCP → Resources
```

---

## 17. Complete IAM + SCP example

Account:

```text
Production Account
```

User:

```text
Alice
```

IAM policy:

```json
{
  "Effect": "Allow",
  "Action": "ec2:*",
  "Resource": "*"
}
```

IAM says:

```text
Alice can use EC2.
```

SCP:

```text
Deny ec2:TerminateInstances
```

Final result:

```text
ec2:RunInstances
    ↓
Allowed

ec2:TerminateInstances
    ↓
Denied
```

So the SCP creates an organizational guardrail even though IAM grants broad EC2 permissions.

---

## 18. Common SCP use cases

### 1. Restrict AWS Regions

Example:

```text
Approved:
ap-south-1
us-east-1

Other Regions:
Restricted
```

Useful for:

- compliance
- data residency
- cost control
- operational consistency

Be careful with global services and AWS services that use global endpoints.

### 2. Protect CloudTrail

Possible guardrails can restrict actions such as:

```text
cloudtrail:StopLogging
cloudtrail:DeleteTrail
cloudtrail:UpdateTrail
```

### 3. Protect production

```text
Production OU
      ↓
Production SCP
      ↓
Production accounts
```

### 4. Restrict prohibited services

Example:

```text
Sandbox OU
      ↓
Deny selected services
```

### 5. Protect organization settings

SCPs can be used to restrict selected organization/account-management actions for member accounts.

---

## 19. SCP architecture

```text
                         AWS ORGANIZATION
                                |
                               ROOT
                                |
              ┌─────────────────┴─────────────────┐
              ↓                                   ↓
        DEVELOPMENT OU                       PRODUCTION OU
              |                                   |
        Development SCP                    Production SCP
              |                                   |
       ┌──────┼──────┐                    ┌───────┼───────┐
       ↓      ↓      ↓                    ↓       ↓       ↓
     Dev-1  Dev-2  Dev-3                Prod-1  Prod-2  Prod-3
       |      |      |                    |       |       |
      IAM    IAM    IAM                  IAM     IAM     IAM
       |      |      |                    |       |       |
       └──────┴──────┘                    └───────┴───────┘
              ↓                                   ↓
       Effective access                    Effective access
```

---

## 20. Production example

A company wants:

> "Production users must not be able to disable CloudTrail."

Architecture:

```text
Production OU
      |
      ↓
Deny-CloudTrail-Changes SCP
      |
 ┌────┼────┐
 ↓    ↓    ↓
Prod1 Prod2 Prod3
```

Even if an IAM policy grants the relevant CloudTrail actions, an applicable explicit deny in the SCP can prevent those actions.

---

## 21. Sandbox example

Sandbox accounts may need more freedom:

```text
Sandbox OU
    ↓
Less restrictive guardrails
```

Production accounts need stricter restrictions:

```text
Production OU
    ↓
Stricter guardrails
```

This is why OU structure and SCP design are closely related.

---

## 22. SCP JSON structure

SCPs use JSON syntax similar to IAM policies.

Basic example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ExampleDeny",
      "Effect": "Deny",
      "Action": [
        "service:Action"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 23. Main SCP elements

| Element | Meaning |
|---|---|
| `Version` | Policy language version |
| `Statement` | Statement container |
| `Sid` | Optional statement identifier |
| `Effect` | `Allow` or `Deny` |
| `Action` | Actions affected |
| `NotAction` | Everything except specified actions |
| `Resource` | Resources affected |
| `NotResource` | Everything except specified resources |
| `Condition` | Conditional logic |

---

## 24. Example — deny one action

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyTerminateInstances",
      "Effect": "Deny",
      "Action": "ec2:TerminateInstances",
      "Resource": "*"
    }
  ]
}
```

Meaning:

```text
IAM may allow EC2 termination
        ↓
SCP denies EC2 termination
        ↓
Final result = DENIED
```

---

## 25. Example — deny multiple actions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyCloudTrailChanges",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "cloudtrail:UpdateTrail"
      ],
      "Resource": "*"
    }
  ]
}
```

This creates a guardrail against those actions.

---

## 26. SCP Conditions

SCPs can use `Condition` to make a restriction dependent on context.

Conceptually:

```text
Deny action
    ↓
Only when a condition is true
```

For example, conditions can be used in Region-based restrictions.

Always test condition logic before production deployment.

---

## 27. SCP and Regions

A common enterprise requirement is:

> "Our workloads should operate only in approved AWS Regions."

Conceptually:

```text
Organization
      |
      ↓
Region Restriction SCP
      |
 ┌────┼────┐
 ↓    ↓    ↓
Dev  Test  Prod
```

The SCP can deny requests outside the approved Region set.

Important:

Global AWS services and global endpoints require special consideration. A Region restriction should be tested against all required operational workflows.

---

## 28. SCP deployment process

A professional workflow:

```text
1. Define the requirement
          ↓
2. Identify the risky action
          ↓
3. Write the SCP
          ↓
4. Validate the JSON
          ↓
5. Test in an isolated OU/account
          ↓
6. Review affected services
          ↓
7. Attach to a small scope
          ↓
8. Monitor
          ↓
9. Expand gradually
          ↓
10. Document
```

Never assume a restrictive SCP is safe just because the JSON is valid.

---

## 29. Testing strategy

Bad approach:

```text
Create SCP
   ↓
Attach to Root
   ↓
Hope everything works
```

Better:

```text
Create test OU
      ↓
Place test account(s)
      ↓
Attach SCP
      ↓
Run workload tests
      ↓
Check AccessDenied events
      ↓
Review CloudTrail / service usage
      ↓
Adjust SCP
      ↓
Deploy gradually
```

AWS recommends thorough testing before applying restrictive SCPs broadly.

---

## 30. SCP troubleshooting

Suppose:

```text
IAM policy:
Allow s3:DeleteBucket

User:
Still receives AccessDenied
```

Check:

```text
1. SCP at Root
2. SCP at OU
3. SCP at Account
4. IAM identity policy
5. Permissions boundary
6. Resource policy
7. Session policy
8. Other applicable authorization controls
```

An applicable SCP deny can be the reason.

---

## 31. Service last accessed data

AWS provides service last accessed information that can help identify which services are actually being used.

Useful workflow:

```text
Account/service usage
       ↓
Service last accessed data
       ↓
Identify unused services
       ↓
Design/refine SCP
       ↓
Test
       ↓
Deploy
```

This helps avoid blocking services that are actually required.

---

## 32. SCP and least privilege

SCPs are one layer in a broader least-privilege design.

```text
IAM
 ↓
Grant only required permissions

SCP
 ↓
Organization-wide maximum guardrail

Permissions Boundary
 ↓
Identity-specific maximum guardrail

Resource Policy
 ↓
Resource access control
```

These controls can work together.

---

## 33. SCP vs Permissions Boundary

| SCP | Permissions Boundary |
|---|---|
| AWS Organizations level | IAM identity level |
| Organization/member-account scope | Specific IAM user/role |
| Central governance | Identity-level governance |
| Managed by Organizations | Managed by IAM |
| Maximum permission guardrail | Maximum permission guardrail |

Memory:

```text
SCP → organization boundary
Permissions Boundary → identity boundary
```

---

## 34. SCP vs IAM Policy vs Resource Policy

| Policy | Main question |
|---|---|
| IAM identity policy | What can this user/role do? |
| Resource policy | Who can access this resource? |
| SCP | What is the maximum this IAM principal can do within the organization? |

Example:

```text
IAM:
Allow Alice to delete an S3 bucket

SCP:
Deny S3 bucket deletion

Result:
Alice cannot delete it.
```

---

## 35. SCP vs RCP

| SCP | RCP |
|---|---|
| Maximum permissions for IAM principals | Maximum permissions for resources |
| Focuses on users/roles in member accounts | Focuses on resources in member accounts |
| Organizational guardrail | Organizational resource guardrail |
| AWS Organizations policy | AWS Organizations policy |

Memory:

```text
SCP → Principal side
RCP → Resource side
```

---

## 36. Limitations

SCPs are powerful, but they are not a complete security solution.

SCPs do not replace:

- IAM
- MFA
- CloudTrail
- monitoring
- network security
- encryption
- vulnerability management
- backup
- incident response

Think:

```text
SCP = one governance/security layer
```

not:

```text
SCP = complete AWS security
```

---

## 37. Billing

Creating and attaching an SCP is not like launching an EC2, NAT Gateway, or RDS database.

There is no separate compute/network resource created by an SCP.

Think:

```text
SCP
 ↓
Organizational policy
 ↓
No workload resource created
```

The actual AWS bill still comes from the services/resources used in the accounts and any applicable AWS pricing/features.

Always verify current AWS pricing for commercial planning.

---

## 38. Common mistakes

### Mistake 1 — Thinking SCP grants permissions

Wrong:

```text
SCP Allow EC2
→ User gets EC2 access
```

Correct:

```text
SCP allows the action at the organizational boundary
+
IAM grants the user/role permission
→ Action can be allowed
```

### Mistake 2 — Wrong OU attachment

A deny attached to the wrong OU can affect many accounts unexpectedly.

### Mistake 3 — Overly broad Deny

For example:

```text
Deny:
service:*
```

can block many required actions.

### Mistake 4 — Dangerous `NotAction`

A `NotAction` statement can have a much wider effect than expected.

### Mistake 5 — Root-level deployment without testing

A Root SCP can have a very broad blast radius.

### Mistake 6 — Assuming SCP controls the management account

It does not restrict users/roles in the management account.

---

## 39. Professional SCP best practices

### 1. Start with a clear requirement

Ask:

> What exactly must be prevented?

### 2. Prefer specific guardrails

Avoid unnecessary broad denies.

### 3. Test before production

Use a test OU/account.

### 4. Use meaningful names

Good:

```text
Deny-Unauthorized-Regions
Deny-CloudTrail-Changes
Deny-Restricted-Services
Deny-Organization-Leave
```

Bad:

```text
Policy1
NewPolicy
ABC
Test2
```

### 5. Document the SCP

For every SCP, document:

```text
Purpose
Scope
Reason
Owner
Exceptions
Dependencies
Testing status
Last review
```

### 6. Deploy gradually

```text
Test
 ↓
Development
 ↓
NonProd
 ↓
Production
 ↓
Broader scope
```

### 7. Monitor after deployment

Check:

- CloudTrail
- AccessDenied events
- application behavior
- AWS service usage
- operational workflows

---

## 40. Example enterprise structure

```text
                         ORGANIZATION ROOT
                                |
              ┌─────────────────┼─────────────────┐
              ↓                 ↓                 ↓
         Security OU        NonProd OU        Production OU
              |                 |                 |
              |            ┌────┴────┐       ┌────┴─────┐
              |            ↓         ↓       ↓          ↓
              |          Dev OU    Test OU  Standard   Critical
              |            |         |       Prod OU    Prod OU
              |            ↓         ↓         |          |
              |         Dev Accts  Test Accts  ↓          ↓
              |                              Prod Accts  Critical
              |
        Security Accounts
```

Possible SCP strategy:

```text
Root
 ↓
Common organization guardrails

Security OU
 ↓
Security-specific restrictions

Sandbox/NonProd
 ↓
Non-production restrictions

Production OU
 ↓
Production protection
```

---

## 41. Final mental model

Think of AWS Organizations like a company:

```text
AWS Organization
        |
        ↓
       Root
        |
        ↓
       OUs
        |
        ↓
     Accounts
        |
        ↓
   IAM Users/Roles
```

Now add SCP:

```text
                 SCP
                  |
                  ↓
       ┌────────────────────┐
       │ Maximum permission │
       │    guardrail       │
       └────────────────────┘
                  |
                  ↓
             IAM User/Role
                  |
                  ↓
             AWS Request
```

The request succeeds only when all applicable authorization controls permit it.

---

## 42. Easy memory trick

Remember:

> **IAM gives permission. SCP limits permission.**

More professional:

```text
IAM → grants permissions
SCP → sets maximum permissions
```

---

## 43. Interview-ready answers

### Q1. What is an SCP?

> An SCP is an AWS Organizations policy that defines the maximum permissions available to IAM users and roles in member accounts.

### Q2. Does SCP grant permissions?

> No. SCPs do not grant permissions. IAM identity-based or resource-based policies still need to grant access.

### Q3. Where can SCPs be attached?

> SCPs can be attached to the organization root, OUs, or individual member accounts.

### Q4. IAM allows an action but SCP denies it. What happens?

> The action is denied because the applicable SCP creates an organizational restriction.

### Q5. Does SCP affect the management account?

> No. SCPs do not restrict users or roles in the management account.

### Q6. Can SCP affect a member account root user?

> Yes, generally, subject to AWS documented exceptions.

### Q7. SCP vs IAM policy?

> IAM policies grant/control access for identities or resources; SCPs define the maximum permissions available to IAM users and roles in member accounts.

### Q8. SCP vs Permissions Boundary?

> SCP is an organization-level guardrail, while a permissions boundary is an IAM-level maximum-permission boundary for a specific user or role.

---

## 44. Final architecture to remember

```text
                         AWS ORGANIZATION
                                |
                               ROOT
                                |
              ┌─────────────────┴─────────────────┐
              ↓                                   ↓
        DEVELOPMENT OU                       PRODUCTION OU
              |                                   |
        Development SCP                    Production SCP
              |                                   |
       ┌──────┼──────┐                    ┌───────┼───────┐
       ↓      ↓      ↓                    ↓       ↓       ↓
     Dev-1  Dev-2  Dev-3                Prod-1  Prod-2  Prod-3
       |      |      |                    |       |       |
      IAM    IAM    IAM                  IAM     IAM     IAM
       |      |      |                    |       |       |
       └──────┴──────┘                    └───────┴───────┘
              ↓                                   ↓
       Effective access                    Effective access
```

Core flow:

```text
SCP
 ↓
Maximum organizational permission
 ↓
IAM policy
 ↓
AWS request
 ↓
Effective authorization
 ↓
ALLOW or DENY
```

---

# 45. Final takeaway

If you remember only five things:

1. **SCP = Service Control Policy**
2. **SCP does NOT grant permissions.**
3. **SCP sets the maximum permissions for IAM users and roles in member accounts.**
4. **SCPs can be attached to the Root, OUs, or accounts.**
5. **IAM provides permissions; SCPs provide organization-level guardrails.**

The simplest professional mental model:

```text
                 ORGANIZATION
                      |
                     SCP
                      ↓
             Maximum Permission
                      |
                     IAM
                      ↓
             Actual Permission
                      |
                      ↓
                AWS Request
                      |
               ┌──────┴──────┐
               ↓             ↓
             ALLOW          DENY
```

---

## Official AWS documentation

- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_syntax.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_policies_create.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_policies_attach.html
