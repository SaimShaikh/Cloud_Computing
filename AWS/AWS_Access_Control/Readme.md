
# 🔐 AWS Access Control Explained 

---

## What is AWS Access Control?

AWS Access Control is a security mechanism that determines **who** can access **what** resources and **what actions** they can perform on those resources. Think of it like a bouncer at a nightclub—they check credentials and decide who gets in and what they can do inside.

Access control ensures that only authorized users, applications, and services can access your AWS resources, protecting your data and infrastructure from unauthorized access.

---

## Why Do We Need Access Control?

- Prevents **data breaches** and abuse of resources
- Ensures only the right people can perform the right actions
- Complies with security standards and regulatory frameworks

---

## When to Use Access Control?

- Always use it for all AWS accounts and resources, not just production
- Especially important for sensitive data, critical workloads, and external access needs

---

## Benefits of Access Control

- Secure isolation between teams and projects
- Enforced compliance (GDPR, HIPAA, PCI-DSS, etc.)
- Minimize damage from accidental or malicious actions
- Clear auditability of "who did what and when"

---

## Types of AWS Access Control

### 🗂️ 1. Identity-Based Policies
*Attach policies to users, groups or roles to define what actions they can perform.*
**Example Scenario:**
A developer needs to manage EC2 instances for development. Attach an identity-based policy granting only EC2 access to that user's IAM identity. They can start/stop EC2 instances but can't access RDS or S3.

---

### 📦 2. Resource-Based Policies
*Attach policies directly to a resource to control who can access it.*
**Example Scenario:**
You have a shared S3 bucket containing media files. Attach a resource-based policy allowing another AWS account or a specific IAM role from another account to read objects, without modifying anything else in your account.

---

### ❌ 3. Explicit Deny
*A policy statement that blocks an action, even if another policy allows it. Deny always wins over allow.*
**Example Scenario:**
Finance users need access to S3 buckets, but all users should be prevented from deleting buckets. Attach an explicit deny statement on `s3:DeleteBucket` in their policy or a shared permission boundary.

---

### 🏛️ 4. Service Control Policies (SCPs)
*Set org-level maximums for permissions across AWS accounts/OUs using AWS Organizations.*
**Example Scenario:**
Your company disables use of expensive AWS regions or experimental services org-wide. Use an SCP to deny access to all regions except `us-east-1` and `us-west-2`. Even with admin permissions, no user in affected accounts can access others.

---

### 🎯 5. Permission Boundaries
*Set the maximum permissions a user or role can have, limiting what delegated admins can do.*
**Example Scenario:**
A DevOps team lead can create or manage IAM users for their team but shouldn't be able to grant admin access. The lead's user creation actions are limited by a permission boundary, so they can't grant any permissions beyond what the boundary defines.

---

### 🔄 6. Session Policies (STS)
*Limit what a user or app can do for the duration of a temporary AWS session.*
**Example Scenario:**
A contractor is hired for a week to troubleshoot logs. They assume a role with STS, and a session policy further restricts their access only to CloudWatch logs with read permissions, for a strictly limited time (e.g., 4 hours per session). When the session expires, access is gone.

---

### 🌐 7. ACLs (Access Control Lists)
*Control legacy or network-level access at the subnet (VPC) or object (S3) level, usually for backward compatibility or fine-grained control.*
**Example Scenario (Network ACL):**
Restrict all incoming traffic to a database subnet except for the corporate VPN IP range, using a network ACL. Even if an EC2 security group allows access, the subnet-level ACL denies all other traffic.

**Example Scenario (S3 Object ACL):**
Legacy workflow grants a specific user in a partner account read-only access to a single S3 object using an S3 ACL, while the rest of the bucket remains private.

---

## Policy Types Comparison Table

| Policy Type | Attached To | Grants Permission? | Scope | Best For |
|------------|-------------|-------------------|-------|----------|
| **Identity-Based** | Users, Groups, Roles | Yes | Single identity | Managing permissions per user/role |
| **Resource-Based** | Resources (S3, SNS, etc.) | Yes | Resource level | Cross-account access, third-party access |
| **Explicit Deny** | Any policy type | No (only denies) | Specific actions | Security guardrails, accident prevention |
| **SCP** | Organization, OUs, Accounts | No (only limits) | All principals in account | Organization-wide compliance |
| **Permission Boundary** | Individual users/roles | No (only limits) | Specific identity | Delegation with safety guardrails |
| **Session Policies** | STS session | No (only limits) | Temporary session | Temporary/federated access |
| **Network ACLs** | VPC subnets | Yes (firewall rules) | Subnet traffic | Network-level traffic control |
| **S3 Object ACLs** | S3 objects | Yes (legacy) | Single object | Legacy S3 access (not recommended) |

---



