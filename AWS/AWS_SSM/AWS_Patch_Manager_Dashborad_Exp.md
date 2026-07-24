# AWS Systems Manager Patch Manager (Quick Setup) - Configuration Guide

## Overview

AWS Systems Manager **Patch Manager** helps automate the process of scanning and installing operating system patches on your managed EC2 and on-premises instances.

Using **Quick Setup**, you can create a Patch Policy that automatically scans, installs, and reports patch compliance across your managed instances.

---

# Patch Policy Name

## What is it?

A Patch Policy Name is simply the name used to identify your patch policy.

It has no impact on how patching works.

### Examples

```text
patch-manager-lab
prod-linux-weekly-patching
dev-linux-patching
windows-monthly-patching
```

### Recommendation

For learning:

```text
patch-manager-lab
```

For production:

```text
prod-linux-weekly-patching
```

---

# Scanning and Installation

This section decides **what Patch Manager should do**.

There are two options.

---

## Option 1 - Scan

### What it does

Patch Manager:

- Checks installed packages
- Compares them with the Patch Baseline
- Finds missing patches
- Generates a compliance report

It **does not install any patches**.

### Workflow

```text
EC2 Instance
      │
      ▼
Check Installed Packages
      │
      ▼
Compare with Patch Baseline
      │
      ▼
Compliance Report
```

### When to choose Scan

Use this option when:

- Production servers
- Compliance audits
- Security assessments
- You want to review missing patches before installing them

### Advantages

- No downtime
- No reboot
- Safe for production
- Compliance reporting

### Disadvantages

- Missing patches remain uninstalled

---

## Option 2 - Scan and Install

### What it does

Patch Manager:

1. Scans the instance
2. Finds missing patches
3. Downloads approved patches
4. Installs them
5. Reboots the instance if required

### Workflow

```text
Scan
   │
   ▼
Find Missing Patches
   │
   ▼
Download
   │
   ▼
Install
   │
   ▼
Reboot (if required)
```

### When to choose Scan and Install

Recommended for:

- Development
- Testing
- Learning AWS
- Lab environments
- Non-production servers

### Advantages

- Fully automated
- Keeps systems updated
- No manual patch installation

### Disadvantages

- May reboot servers
- Can temporarily interrupt applications

---

# Scanning Schedule

This determines **when Patch Manager performs the scan or installation**.

---

## Option 1 - Use Recommended Defaults

AWS automatically schedules the patch operation.

Default schedule:

```text
Every day
1:00 AM UTC
```

### Best for

- Learning
- Labs
- Small environments
- Users who don't require custom schedules

---

## Option 2 - Custom Scan Schedule

You define your own schedule.

Examples

Every Sunday

```text
2:00 AM
```

Every Month

```text
First Saturday
3:00 AM
```

Every Friday

```text
11:00 PM
```

### Wait to Scan Targets Until First CRON Interval

If enabled:

Patch Manager waits until the first scheduled CRON time before running.

Example

Schedule:

```text
Every Sunday 2 AM
```

Policy created on Wednesday.

Result:

Nothing happens immediately.

The first scan starts on Sunday at 2 AM.

If disabled:

Patch Manager performs its first scan immediately after deployment and then follows the schedule.

### Best for

Production environments that follow maintenance windows.

---

# Patch Baseline

## What is a Patch Baseline?

A Patch Baseline is a collection of rules that tells AWS:

- Which patches are approved
- Which patches are rejected
- How many days after release a patch becomes approved
- Which patch classifications are allowed

### Workflow

```text
AWS Releases Patch
        │
        ▼
Patch Baseline
        │
 ┌──────┴──────┐
 │             │
 ▼             ▼
Approved    Rejected
 │             │
 ▼             ▼
Install    Skip
```

---

## Option 1 - Use Recommended Defaults

AWS uses the default Patch Baseline for each operating system.

Examples

| Operating System | Default Baseline |
|------------------|------------------|
| Amazon Linux | AWS-AmazonLinuxDefaultPatchBaseline |
| Amazon Linux 2 | AWS-AmazonLinux2DefaultPatchBaseline |
| Amazon Linux 2023 | AWS-AmazonLinux2023DefaultPatchBaseline |
| Ubuntu | AWS-UbuntuDefaultPatchBaseline |
| RHEL | AWS-RedHatDefaultPatchBaseline |
| Windows | AWS-DefaultPatchBaseline |

AWS automatically maintains these baselines.

### Best for

- Learning
- Labs
- Small businesses
- Most AWS users

---

## Option 2 - Custom Patch Baseline

Create your own approval rules.

Example

```text
Security Updates
Approve after 2 days

Critical Updates
Approve after 1 day

Bug Fixes
Approve after 14 days

Feature Updates
Never approve automatically
```

You can also:

- Reject specific patches
- Approve only selected patches
- Delay installations

### Best for

- Enterprises
- Banking
- Healthcare
- Government
- Compliance requirements

---

# Patching Log Storage

Patch Manager stores execution logs in an S3 bucket.

Example

```text
Patch Started

↓

Downloaded Kernel

↓

Installed OpenSSL

↓

Restart Required

↓

Logs Stored in Amazon S3
```

### Benefits

- Troubleshooting
- Patch history
- Compliance evidence
- Long-term log storage

### Recommendation

Keep S3 logging enabled.

---

# Target Region

Defines where the Patch Policy will be deployed.

### Current Region

Deploys only to the selected AWS Region.

Example

```text
eu-north-1
```

Only managed instances in that region receive the policy.

### Multiple Regions

You can deploy one Patch Policy to multiple AWS Regions.

Example

```text
eu-north-1
ap-south-1
us-east-1
```

### Recommendation

For labs:

Use **Current Region**.

---

# Target Nodes

Determines which managed instances receive the Patch Policy.

---

## Option 1 - All Managed Nodes

Every managed EC2 instance receives the Patch Policy.

Example

```text
EC2-1

EC2-2

EC2-3

EC2-4
```

All instances are patched.

### Best for

- Labs
- Small environments

---

## Option 2 - Specify Node Tag

Only instances with matching tags receive the policy.

Example

```text
Environment = Production
```

Only production servers are patched.

Another example

```text
PatchGroup = Linux
```

Only Linux servers receive the policy.

### Best for

Production environments.

---

# Rate Control

Rate Control prevents patching every server simultaneously.

---

## Concurrency

Defines how many instances can be patched at the same time.

Current Example

```text
10%
```

If there are 100 instances:

```text
10 instances patched

↓

Next 10

↓

Next 10
```

instead of all 100 at once.

### Why use Concurrency?

- Prevents service outages
- Reduces infrastructure load
- Maintains application availability

### Recommendation

Lab

```text
10%
```

Production

```text
10%–20%
```

depending on business requirements.

---

## Error Threshold

Defines how many failures are allowed before Patch Manager stops.

Example

```text
100 Instances

↓

2 Instances Fail

↓

Continue

↓

3 Instances Fail

↓

Stop Patching
```

### Why use Error Threshold?

If many servers begin failing, Patch Manager stops instead of risking all systems.

### Recommendation

Lab

```text
2%
```

Production

```text
5%–10%
```

depending on company policy.

---

# Instance Profile Options

This option allows Quick Setup to automatically update existing EC2 IAM roles.

If enabled, AWS automatically adds these IAM policies when needed:

```text
AmazonSSMManagedInstanceCore

aws-quicksetup-patchpolicy-baselineoverrides-s3
```

Without these permissions, Patch Manager cannot successfully patch instances.

### Enabled

- Automatically updates IAM instance profiles
- Less manual work
- Recommended for most environments

### Disabled

You must manually attach the required IAM permissions.

If forgotten, patching operations will fail.

### Recommendation

Enable this option.

---

# Local Deployment Roles

Quick Setup requires IAM roles to deploy and manage the Patch Policy.

---

## Create and Use New IAM Roles

AWS automatically creates:

```text
AWS-QuickSetup-PatchPolicy-LocalAdministration

AWS-QuickSetup-PatchPolicy-LocalExecutionRole
```

Recommended for:

- First-time setup
- Labs
- Learning
- Small environments

---

## Use Existing IAM Roles

Choose this only if your organization already has approved deployment roles.

Typically used in enterprise environments.

### Recommendation

Create and use new IAM deployment roles.

---

# Recommended Configuration for Learning

| Setting | Recommended Value |
|----------|-------------------|
| Patch Policy Name | patch-manager-lab |
| Patch Operation | Scan and Install |
| Scanning Schedule | Use Recommended Defaults |
| Patch Baseline | Use Recommended Defaults |
| Patching Logs | Enable S3 Logging |
| Target Region | Current Region |
| Target Nodes | All Managed Nodes |
| Concurrency | 10% |
| Error Threshold | 2% |
| Instance Profile Options | Enable |
| Local Deployment Roles | Create New IAM Roles |

---

# Production Best Practices

| Setting | Recommended |
|----------|-------------|
| Patch Operation | Scan |
| Scan Schedule | Custom Maintenance Window |
| Patch Baseline | Custom Patch Baseline |
| Target Nodes | Tag-Based Targeting |
| Concurrency | 10%–20% |
| Error Threshold | 5%–10% |
| Patching Logs | Enable S3 Logging |
| IAM Roles | Use Existing Organization Roles |

---

# Key Takeaways

- **Patch Policy** defines how and when patching occurs.
- **Patch Operation** controls whether AWS only scans or also installs patches.
- **Scanning Schedule** determines when patching runs.
- **Patch Baseline** decides which patches are approved for installation.
- **Target Nodes** specifies which instances receive the policy.
- **Rate Control** limits simultaneous patching and controls acceptable failures.
- **S3 Logging** stores detailed patch execution logs.
- **Instance Profile Options** ensure required IAM permissions are available.
- **Local Deployment Roles** allow Quick Setup to deploy and manage the patch policy.

For learning and lab environments, the AWS recommended defaults provide a simple, automated setup. In production, organizations typically use custom schedules, custom patch baselines, tag-based targeting, and controlled maintenance windows to minimize business impact.
