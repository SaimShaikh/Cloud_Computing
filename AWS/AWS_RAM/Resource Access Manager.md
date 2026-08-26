# AWS Resource Access Manager (RAM) — Simple Guide

## 1. What It Is

AWS RAM lets one AWS account **share resources** (like a subnet or Transit Gateway) with other accounts, OUs, or your whole Organization — without copying the resource and without giving up ownership.

```
Owner Account
     │
   Resource (e.g. Transit Gateway)
     │
     │  shared via RAM
     ▼
Consumer Account
     │
   Can use it — doesn't own it
```

No duplicate infrastructure. No cross-account IAM plumbing per resource. Ownership and billing always stay with the owner.

---

## 2. Core Components

```
                Resource Share
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
     Resource       Principal      Permission
     "WHAT?"        "WHO?"         "WHAT CAN?"
```

| Term | Meaning |
|---|---|
| Resource | What you're sharing (subnet, TGW, IPAM pool, etc.) |
| Principal | Who gets access — an AWS account, an OU, or the whole Organization |
| Managed Permission | What the principal is allowed to do with it |
| Resource Share | The object that ties all three together |
| Owner Account | Owns and pays for the resource — this never changes |
| Consumer Account | Gets usage rights only |

---

## 3. AWS Managed vs. Customer Managed Permissions

| | AWS Managed | Customer Managed |
|---|---|---|
| Who creates it | AWS | You |
| Customizable | No, fixed | Yes |
| Use case | Standard sharing | Fine-grained/least-privilege access |
| Available for | Every RAM-supported type | Only some resource types |

Effective access is always the overlap of **RAM permission ∩ IAM policy ∩ SCP** — an SCP deny always wins, even if RAM and IAM both allow it. (More on SCPs in §4.)

---

## 4. SCP (Service Control Policy) — What, Why, How, Limitations

### What it is
A Service Control Policy is an **AWS Organizations-level guardrail**. It's a JSON policy attached to the Organization root, an OU, or an individual account that defines the **maximum available permissions** for every principal in that account — including the root user.

```
AWS Organization
      │
      ▼
    SCP attached at root / OU / account
      │
      ▼
Sets the ceiling of what ANY IAM user/role in that account can ever do
```

SCPs don't grant permissions by themselves. They only **filter** what IAM can grant. Think of it as a permissions ceiling, not a permissions grant.

### Why it exists
Without SCPs, individual account admins (or a compromised IAM role) could do anything IAM allows — including things that violate company policy, like disabling CloudTrail, opening a Region you don't use, or deleting backups. SCPs let a central security/platform team enforce guardrails **org-wide**, regardless of what any single account's IAM policies say.

Typical reasons teams use SCPs:
- Block use of Regions outside an approved list
- Prevent disabling of CloudTrail, GuardDuty, Config
- Block root user actions
- Enforce that resources can only be created with required tags
- Prevent leaving the AWS Organization
- **Control what can be shared or accepted via RAM** (see below)

### How it works

```
Request to perform an action
         │
         ▼
   Does an applicable SCP explicitly ALLOW it (or not deny it)?
         │
   ┌─────┴─────┐
  YES          NO (explicit deny, or not in an Allow list)
   │             │
   ▼             ▼
Continue to    ❌ Denied — IAM is never even evaluated
IAM evaluation
```

Two SCP strategies:
| Strategy | How it works |
|---|---|
| **Deny list** (default AWS-managed `FullAWSAccess` stays attached) | Start from full access, attach specific `Deny` statements for what's forbidden |
| **Allow list** | Remove `FullAWSAccess`, explicitly `Allow` only approved services/actions — much stricter |

Example deny SCP relevant to RAM:
```json
{
  "Effect": "Deny",
  "Action": [
    "ram:AcceptResourceShareInvitation"
  ],
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "ram:ShareOwnerAccountId": "111111111111"
    }
  }
}
```
This blocks any account under the SCP from accepting a RAM invitation from any owner **except** the trusted Network Account — a common guardrail to stop shadow IT from accepting shares from unknown external accounts.

### How SCPs interact with RAM specifically
```
RAM Permission   → what the resource share allows
IAM Policy       → what the consumer's role/user is allowed to call
SCP              → org-wide ceiling, evaluated first, can only restrict
──────────────────────────────────────────────
Effective access = RAM ∩ IAM ∩ SCP
```
Even if a resource share grants full access and the consumer's IAM role allows it, an SCP `Deny` on that action blocks it completely. SCPs are evaluated **before** IAM policies — if the SCP denies it, IAM is never even consulted.

### Limitations
- **SCPs never grant permissions** — attaching an `Allow` SCP does nothing if the IAM policy in the account doesn't also allow the action; SCPs only narrow, never widen.
- **Don't apply to the AWS Organizations management account** — SCPs affect member accounts only, not the account at the top of the org.
- **Don't apply to service-linked roles** in most cases — some AWS-managed roles are exempt so core services keep functioning.
- **Can't be resource-specific in older/simple setups the way IAM resource policies can** — SCPs work at the action/condition level, not fine-grained per-object unless you use condition keys (like the `ram:ShareOwnerAccountId` example above).
- **Org-wide blast radius** — a badly written SCP can lock an entire OU out of a needed action; always test in a sandbox OU first.
- **Size and quota limits** — SCPs have a maximum document size and a limit on how many can be attached per OU/account, so very granular per-team rules can hit quota limits at scale.
- **No visibility into "why was this denied" from the consumer side** — a denied API call just returns `AccessDenied`; you have to check CloudTrail or the SCP simulator to know it was the SCP, not IAM or RAM, that blocked it.

---

## 5. Region Rules

- RAM is a **Regional service**. A resource share for a Regional resource (like a subnet) must be created in that same Region.
- You **cannot** combine resources from two different Regions in one share.
- **Global resources** (no Region in their ARN) always use `us-east-1` as RAM's designated share Region — this only affects where the share object lives, not where the resource itself runs.
- A shared resource stays in its own Region — the consumer account doesn't get a copy elsewhere.

---

## 6. Commonly Shared Resource Types

| AWS Service | RAM-shareable resources |
|---|---|
| Amazon VPC | Subnets, Prefix Lists, Transit Gateways, IPAM pools, Route 53 Resolver rule associations, Network Firewall resources, etc. |
| AWS Transit Gateway | Transit Gateways |
| AWS Network Firewall | Firewall policies, rule groups |
| Route 53 | Resolver rules / related supported resources |
| VPC Lattice | Services, Service Networks, Resource Configurations |
| EC2 | Capacity Reservations, Dedicated Hosts-related resources, Local Gateway Route Tables, etc. |
| AWS Cloud WAN | Core Networks |
| AWS Outposts | Outposts, Sites, Local Gateway Route Tables |
| Amazon S3 | S3 Access Grants |
| Amazon EFS / FSx | Supported FSx volumes/resources |
| AWS Private CA | Private Certificate Authorities |
| AWS Backup | Backup Vaults |
| AWS Cloud Map | Namespaces |
| AWS App Mesh | Meshes |
| AWS AppSync | GraphQL APIs |
| API Gateway | Private Custom Domains |
| Amazon Aurora | DB Clusters |
| Amazon Bedrock | Custom Models |
| AWS CodeBuild | Projects, Report Groups |
| AWS CodeConnections | Connections |
| Amazon DataZone | Domains |
| EC2 Image Builder | Components, Images, Image Recipes, Container Recipes |
| Elastic Load Balancing | Trust Stores |
| AWS Glue | Data Catalog resources |
| AWS License Manager | License Configurations |
| AWS Marketplace | Marketplace Catalog Entities |
| AWS Resource Explorer | Views |
| AWS Resource Groups | Resource Groups |
| AWS Service Catalog AppRegistry | Applications, Attribute Groups |
| Amazon CloudFront | VPC Origins |
| AWS CloudHSM | Backups |
| AWS End User Messaging SMS | Opt-out Lists |
| AWS Billing / Cost Management | Billing Views, BCM Dashboards |
| AWS SageMaker AI | Supported SageMaker resources |
| AWS Network Manager / Cloud WAN | Core Network resources |
| AWS Multi-party Approval | Approval Teams |

Get the current full list anytime:
```bash
aws ram list-resource-types
```

Note: not everything uses RAM. S3, KMS, DynamoDB, and Lambda use bucket/resource policies instead — RAM is specifically for the resource types listed above.

---

## 7. How to Share a Resource (Console)

**Owner account:**
1. RAM Console → confirm Region matches the resource's Region
2. **Create resource share** → name it
3. Select the resource(s) to share
4. Select a managed permission (AWS default, or a custom one)
5. Add principal(s) — account ID, OU, or organization
6. Create

**Consumer account:**
1. If org-wide auto-accept is on, the resource just appears
2. Otherwise: RAM → **Shared with me** → accept the pending invitation

---

## 8. CLI Quick Reference

```bash
# Create a share
aws ram create-resource-share \
  --name my-share \
  --resource-arns arn:aws:ec2:ap-south-1:111111111111:subnet/subnet-0abc12345 \
  --principals 222222222222 \
  --permission-arns arn:aws:ram::aws:permission/AWSRAMDefaultPermissionSubnet

# Accept a share (consumer side)
aws ram get-resource-share-invitations
aws ram accept-resource-share-invitation --resource-share-invitation-arn <arn>

# Add/remove resources or principals
aws ram associate-resource-share --resource-share-arn <arn> --resource-arns <arn>
aws ram disassociate-resource-share --resource-share-arn <arn> --resource-arns <arn>

# Delete a share
aws ram delete-resource-share --resource-share-arn <arn>

# List supported resource types / current shares
aws ram list-resource-types
aws ram get-resource-shares --resource-owner SELF
```

---

## 9. Billing — Key Point

**AWS RAM itself is free.** No charge for creating shares, adding resources/principals, or accepting invitations.

All real cost comes from the **underlying resource**:

| Resource | Who pays |
|---|---|
| Transit Gateway itself | Owner account |
| TGW attachment + data processed | Whoever creates the attachment (usually the consumer) |
| Shared subnet | Free — cost comes from what's launched into it (billed to whoever launches it) |
| Private CA monthly fee & per-cert fees | Always the owner account, even for certs issued on a consumer's behalf |
| IPAM pool | Owner account |

> RAM only changes **who can use** a resource — never **who pays** for it. If you need to attribute shared-resource cost back to a team, use Cost Allocation Tags or Cost Categories; RAM itself has no billing signal.

---

## 10. Common Issues

| Problem | Cause | Fix |
|---|---|---|
| Consumer can't see the shared resource | Invitation still pending | Accept it via `get-resource-share-invitations` / `accept-resource-share-invitation` |
| Can't create a global-resource share in another Region | Global shares must be created in `us-east-1` | Switch Region to `us-east-1` |
| "Resources from different Regions" error | Tried mixing Regions in one share | Split into one share per Region |
| Consumer sees resource but action fails | RAM permission, IAM, or SCP is blocking it | Check the intersection of all three |
| Permission change didn't affect an existing share | Shares pin the permission version at creation | Explicitly update the share to the new version |
| Action denied with no clear reason | Likely an SCP deny, evaluated before IAM | Check CloudTrail event / SCP simulator to confirm |

---

## 11. Interview-Ready Q&A

**Does RAM transfer ownership?** No — ownership and billing always stay with the owner.

**Is RAM itself billed?** No — it's free; cost comes only from the shared resource's usage.

**Can one share span two Regions?** No — Regional resources must be shared from their own Region.

**Where do global-resource shares live?** Always `us-east-1`.

**AWS managed vs. customer managed permission?** AWS managed is fixed and AWS-maintained; customer managed is authored and versioned by you, where the resource type supports it.

**Does an SCP override RAM?** Yes — SCPs can only restrict, never grant; effective access is the intersection of RAM, IAM, and SCP.

**What's the #1 thing people get wrong about SCPs?** Assuming an `Allow` SCP grants access — it doesn't. SCPs only set a ceiling; IAM still has to separately grant the permission.

---

## 12. Cheat Sheet

```
Resource + Principal + Permission → Resource Share
Owner keeps ownership & billing. Consumer gets usage rights only.
RAM = free. Resource usage = normal AWS pricing.
Regional resource → share lives in its own Region.
Global resource → share always lives in us-east-1.
SCP = org-wide ceiling. Never grants, only restricts. Evaluated before IAM.
Effective access = RAM ∩ IAM ∩ SCP.
```

---

## 13. Cleanup Order

1. Remove consumer-side usage (e.g., delete the TGW attachment, terminate instances)
2. Disassociate resources/principals from the share
3. Delete the resource share
4. Delete any custom permissions no longer needed
5. Delete the underlying resource in the owner account (only once nothing depends on it)
