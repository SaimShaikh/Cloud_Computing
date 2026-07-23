# AWS Systems Manager Automation - Change EC2 Instance Type

> **Objective:** Learn how to automate changing an EC2 instance type (instance size) using **AWS Systems Manager Automation**, instead of manually resizing through the EC2 console.

---

## Table of Contents

1. Introduction
2. Why Use SSM Automation?
3. How It Works
4. Architecture
5. Prerequisites
6. IAM Permissions and Automation Role
7. Create the Automation Document
8. Execute the Automation
9. Monitor the Execution
10. Verify the Resize
11. Schedule Using a Maintenance Window
12. Troubleshooting

---

## 1. Introduction

Normally, changing an EC2 instance type is a manual, three-step process:

- Stop the EC2 instance
- Change the instance type
- Start the EC2 instance

That's fine for one or two servers, but becomes slow and error-prone at scale, or when it has to happen on a recurring schedule.

**AWS Systems Manager (SSM) Automation** removes the manual work by running these steps as an Automation Document (a "runbook"). Instead of you clicking through the console, SSM calls the required EC2 APIs on your behalf.

---

## 2. Why Use SSM Automation?

**Manual process**

```
Administrator → EC2 Console → Stop Instance → Change Instance Type → Start Instance
```

**Automated process**

```
Administrator → Systems Manager → Automation Document → AWS performs all steps → Instance Running
```

**Benefits**

- Fully automated, repeatable, and auditable
- Can resize many instances at once instead of one at a time
- Removes manual console clicks from production changes
- Can run on a schedule (e.g., off-peak resizing)
- Every execution is logged, so you get a clear audit trail

---

## 3. How It Works

Systems Manager Automation doesn't change the instance type directly — there's no single "resize" API. Instead, it orchestrates a sequence of existing EC2 API calls:

```
Stop Instance → Modify Instance Type → Start Instance → Verify Running State
```

Internally:

```
StopInstances() → ModifyInstanceAttribute() → StartInstances() → DescribeInstances()
```

`ModifyInstanceAttribute` is the actual API that changes the instance type — it can only be called while the instance is stopped.

---

## 4. Architecture

```
                    Administrator
                          │
                          ▼
          AWS Systems Manager Automation
                          │
                 Automation Document
                          │
        ┌─────────────────┼──────────────────┐
        ▼                 ▼                  ▼
 Stop EC2 Instance   Change Instance Type   Start EC2 Instance
        │                 │                  │
        └─────────────────┼──────────────────┘
                          ▼
                EC2 Running Successfully
```

---

## 5. Prerequisites

**EC2 instance**
- Running EC2 instance with an EBS-backed root volume (Instance Store-backed instances can't be resized this way — see the FAQ in Section 14)
- Supported operating system for the target instance type

**Systems Manager**
- The instance must appear under **Systems Manager → Managed Nodes** with status **Online**

**SSM Agent**
- Must be installed and running. Most current Amazon Linux, Ubuntu, and Windows AMIs include it by default — older/custom AMIs may need it installed manually.

**IAM role on the EC2 instance**
- Attach the managed policy `AmazonSSMManagedInstanceCore` so the instance can communicate with Systems Manager.

**Network connectivity**
- The instance needs outbound HTTPS (TCP 443) to Systems Manager, via:
  - Internet Gateway + NAT Gateway, **or**
  - Systems Manager Interface VPC Endpoints (`ssm`, `ssmmessages`, `ec2messages`) if it's in a fully private subnet

---

## 6. IAM Permissions and Automation Role

This is the step where automations most commonly fail, so it's worth getting exactly right.

### What the Automation execution role needs

The automation runs as an IAM role (an "Automation Assume Role"), separate from the role on the EC2 instance itself. That role needs permission to:

```
ec2:StopInstances
ec2:StartInstances
ec2:ModifyInstanceAttribute
ec2:DescribeInstances
```

### Important: the AWS managed policy alone is not enough

AWS provides a managed policy called `AmazonSSMAutomationRole` for automation roles. It's a reasonable starting point, but **it does not include `ec2:ModifyInstanceAttribute` or `ec2:DescribeInstances`** — the two permissions this specific automation depends on most. If you attach only `AmazonSSMAutomationRole` and skip the custom policy, the automation will get through "Stop Instance" and then fail at the "Modify Instance Type" step.

So attaching a custom inline policy with the four actions above isn't optional here — it's required for this use case.

### Create the role

1. Open **IAM → Roles → Create Role**
2. Trusted entity: **AWS Service**
3. Use case: **Systems Manager**
4. Attach the managed policy `AmazonSSMAutomationRole`
5. Add an inline policy granting:
   ```
   ec2:StopInstances
   ec2:StartInstances
   ec2:ModifyInstanceAttribute
   ec2:DescribeInstances
   ```
6. Name it, e.g., `SSM-Automation-Resize-Role`, and create it.

---

## 7. Create the Automation Document

AWS does publish some resize-related runbooks (for example, `AWSPremiumSupport-ResizeNitroInstance` and `AWSPremiumSupport-ChangeInstanceTypeIntelToAMD`), but these are Premium/Enterprise Support-tier runbooks — they aren't available in every account, and they're built for specific migration scenarios rather than general-purpose resizing. For a portfolio lab (or a general production use case), writing your own document is the more reliable and reusable approach.

Open **Systems Manager → Documents → Create Document → Automation**

Example name: `Resize-EC2-Instance`

The document should:

1. Accept the EC2 Instance ID as input.
2. Accept the new instance type as input.
3. Stop the EC2 instance.
4. Call `ModifyInstanceAttribute` to set the new instance type.
5. Start the EC2 instance.
6. Wait until the instance reaches the running state.

**Automation workflow**

```
Input: Instance ID + New Instance Type
        ↓
   Stop Instance
        ↓
  Modify Instance Type
        ↓
   Start Instance
        ↓
      Wait
        ↓
     Success
```

---

## 8. Execute the Automation

Open **Systems Manager → Automation → Execute Automation**, select `Resize-EC2-Instance`, and provide:

| Parameter | Example |
|---|---|
| Instance ID | `i-0123456789abcdef0` |
| New Instance Type | `t3.medium` |
| Automation Role | `SSM-Automation-Resize-Role` |

Click **Execute**.

---

## 9. Monitor the Execution

Open **Systems Manager → Automation → Executions** and watch the step progress:

```
✓ Stop Instance
✓ Modify Instance
✓ Start Instance
✓ Verify Running
✓ Completed
```

If a step fails, the execution details show exactly which step failed and why — check this before re-running.

---

## 10. Verify the Resize

Open **EC2 → Instances** and confirm the **Instance Type** column now shows the new type (e.g., `t3.medium`).

Connect using Session Manager, SSH, or RDP and confirm the application is behaving correctly under the new instance size.

---

## 11. Schedule Using a Maintenance Window

To run resizes on a recurring schedule instead of manually:

1. Open **Systems Manager → Maintenance Windows** and create one (e.g., every Sunday at 02:00 AM).
2. Register the Automation document as a task within that maintenance window, passing in the target instance and instance type as parameters.

Systems Manager will then trigger the automation automatically at the scheduled time.

---

## 12. Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| EC2 not showing in Managed Nodes | SSM Agent not running, wrong IAM role, or no network path to SSM | Verify SSM Agent status, confirm `AmazonSSMManagedInstanceCore` is attached, check port 443 connectivity / VPC endpoints |
| Automation fails at "Modify Instance Type" | Automation role is missing `ec2:ModifyInstanceAttribute` or `ec2:DescribeInstances` | Attach the custom inline policy from Section 6 — `AmazonSSMAutomationRole` alone doesn't cover this |
| Automation fails at "Start Instance" | Selected instance type not supported in that Availability Zone | Choose a compatible instance type, or move the instance to an AZ that supports it |
| "Insufficient capacity" error | AWS has no current capacity for the requested type in that AZ | Retry later, or pick another instance type/family |
| Instance doesn't start after resize | Unsupported OS/architecture for the new type, or an underlying boot issue | Check EC2 system log and status checks; confirm the AMI supports the new instance family (e.g., Nitro vs. Xen) |

---

