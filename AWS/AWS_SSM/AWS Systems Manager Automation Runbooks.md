# AWS Systems Manager Automation Runbooks — Complete Deep Dive + Hands-On Lab

> A scenario-driven, in-depth guide to SSM Automation runbooks: how they work internally, how to author them, and a full step-by-step hands-on lab building a real "Stop → Snapshot → Resize → Start → Verify" runbook end to end.

---

## Table of Contents

1. Scenario
2. What is an SSM Automation Runbook?
3. Why Runbooks Matter — The Problem They Solve
4. AWS-Owned Runbooks vs Custom Runbooks
5. Anatomy of an Automation Document
6. Common Automation Action Types
7. How Automation Execution Works Internally
8. Execution Modes — Single Target vs Rate Control (Fleet-Wide)
9. Approval Workflows Inside a Runbook
10. Hands-On Lab — Build a "Stop → Snapshot → Resize → Start → Verify" Runbook
11. Real-World Scenarios and Use Cases
12. Edge Cases and Failure Scenarios
13. Troubleshooting Table
14. Security Considerations
15. Interview Questions
16. Cheat Sheet
17. Mastery Checklist
18. Cleanup Steps

---

## 1. Scenario

Your team manages a fleet of EC2 instances whose root EBS volumes keep running out of space as log files and application data grow. Today, whenever a volume needs resizing, an engineer manually:

1. Logs into the AWS Console
2. Stops the instance
3. Takes a manual snapshot "just in case"
4. Modifies the volume size
5. Starts the instance back up
6. Logs in and manually extends the partition/filesystem
7. Checks that the instance is healthy again

This works, but it is slow, inconsistent between engineers, undocumented, and entirely dependent on one person remembering every step correctly — especially the "take a snapshot first" safety step, which is the one most often skipped under time pressure.

Leadership wants this turned into a **repeatable, self-documenting, auditable procedure** that any engineer (or an automated trigger) can run with a single click and one input: the Instance ID.

This is exactly the problem SSM **Automation runbooks** solve — and this guide builds that exact runbook from scratch, step by step.

---

## 2. What is an SSM Automation Runbook?

An SSM Automation runbook is a **document** (written in YAML or JSON) that defines a sequence of steps — each step calling an AWS API, running a script, waiting for a condition, or pausing for human approval — that Systems Manager executes on your behalf, in order, with built-in retry, failure handling, and a full execution history.

Think of it as **infrastructure-as-code for operational procedures**, not for provisioning. Terraform/CloudFormation define what your infrastructure *is*; an Automation runbook defines what to *do* to that infrastructure, safely and repeatably, on demand or on a trigger.

---

## 3. Why Runbooks Matter — The Problem They Solve

### Without runbooks

```
Incident happens
      │
      ▼
Engineer opens a wiki page / Slack thread from last time
      │
      ▼
Engineer manually runs 7 steps in the console
      │
      ▼
One step gets skipped under pressure (e.g., the snapshot)
      │
      ▼
Outcome depends entirely on who was on call
```

### With runbooks

```
Incident happens (or scheduled trigger fires)
      │
      ▼
Automation runbook starts (console click, CLI, or EventBridge rule)
      │
      ▼
Every step runs in the exact same order, every time
      │
      ▼
Full execution log stored automatically — no wiki page needed
      │
      ▼
Outcome is identical regardless of who or what triggered it
```

Key benefits:

- **Consistency** — the same steps run the same way every time, regardless of who (or what system) triggers it.
- **Auditability** — every execution, every step's input/output, and every failure is logged and viewable in the console and CloudTrail.
- **Safety** — built-in `onFailure` behavior (Abort / Continue / step branching) prevents a failed step from silently cascading into a worse outcome.
- **Reusability** — one runbook, parameterized, works across every instance in the fleet instead of being rewritten per-incident.
- **Automatable** — the same runbook a human clicks manually can also be triggered automatically by an EventBridge rule reacting to a CloudWatch alarm.

---

## 4. AWS-Owned Runbooks vs Custom Runbooks

| Type | Description | Example |
|---|---|---|
| AWS-Owned | Pre-built, maintained by AWS, covers common operational tasks | `AWS-RestartEC2Instance`, `AWS-RunPatchBaseline`, `AWS-CreateSnapshot`, `AWS-StopEC2Instance` |
| Custom | Authored by you, tailored to your exact environment and process | The Stop → Snapshot → Resize → Start → Verify runbook built in this lab |
| Shared | Custom runbooks shared across accounts/organizations via Resource Access Manager | A central platform team's approved "safe resize" runbook shared to every business unit account |

AWS-owned runbooks are a good starting point, but real operational value comes from **custom runbooks** that encode your team's specific safety steps (like "always snapshot before resizing") — which is exactly what AWS-owned generic actions don't know to do for you.

---

## 5. Anatomy of an Automation Document

Every Automation document has the same skeleton:

```yaml
description: "Human-readable description of what this runbook does"
schemaVersion: '0.3'
assumeRole: '{{ AutomationAssumeRole }}'

parameters:
  InstanceId:
    type: String
    description: "(Required) The instance to operate on"
  AutomationAssumeRole:
    type: String
    description: "(Required) IAM role ARN Automation assumes to perform actions"
    default: ''

mainSteps:
  - name: stepOneName
    action: 'aws:someActionType'
    onFailure: Abort
    inputs:
      SomeKey: '{{ InstanceId }}'
    outputs:
      - Name: someOutputVariable
        Selector: '$.SomePath.InResponse'
        Type: String
    nextStep: stepTwoName

  - name: stepTwoName
    action: 'aws:anotherActionType'
    isEnd: true
    inputs:
      AnotherKey: '{{ stepOneName.someOutputVariable }}'
```

### Key fields explained

| Field | Purpose |
|---|---|
| `schemaVersion` | Almost always `'0.3'` for modern Automation documents |
| `assumeRole` | The IAM role Automation assumes to call AWS APIs on your behalf — this is what gives the runbook its permissions, separate from whoever clicks "Execute" |
| `parameters` | Inputs collected before execution starts (Instance ID, desired size, approver ARN, etc.) |
| `mainSteps` | The ordered list of actions that make up the runbook |
| `action` | The action type for that step (see section 6) |
| `inputs` | Parameters passed into that specific action |
| `outputs` | Named values extracted from the action's response, referenceable in later steps as `{{ stepName.outputName }}` |
| `nextStep` | Explicit control-flow — which step runs next (enables branching logic) |
| `onFailure` | What happens if the step fails: `Abort` (stop the whole automation), `Continue` (move to next step anyway), or a named step to jump to |
| `isEnd` | Marks a step as the final step in that branch of execution |

---

## 6. Common Automation Action Types

| Action | Purpose |
|---|---|
| `aws:executeAwsApi` | Call almost any read or write AWS API directly (e.g., `DescribeInstances`, `CreateSnapshot`, `ModifyVolume`) |
| `aws:changeInstanceState` | Start, stop, terminate, or reboot an EC2 instance and optionally wait for the state |
| `aws:waitForAwsResourceProperty` | Poll an API repeatedly until a property reaches a desired value (e.g., wait until a snapshot's `State` becomes `completed`) |
| `aws:runCommand` | Invoke a Run Command document (like `AWS-RunShellScript`) inside the runbook — used for OS-level actions such as extending a filesystem |
| `aws:executeScript` | Run an inline Python or PowerShell script for logic AWS APIs alone can't express |
| `aws:approve` | Pause execution and wait for a human approver (via SNS notification) before continuing |
| `aws:branch` | Conditional branching to different next steps based on a comparison |
| `aws:sleep` / `aws:pause` | Wait a fixed duration, or pause indefinitely until manually resumed |
| `aws:createStack` | Trigger a CloudFormation stack as part of the automation |
| `aws:invokeLambdaFunction` | Call a Lambda function for custom logic and return its output into the runbook |

---

## 7. How Automation Execution Works Internally

```
Trigger
 (Console click / CLI / EventBridge rule / Maintenance Window)
      │
      ▼
Systems Manager Automation Service
      │
      ▼
Assumes the IAM role in `assumeRole`
      │
      ▼
Executes mainSteps in order (or per nextStep branching)
      │
   ┌──┴───────────────┐
   │                  │
 Step calls AWS API   Step calls SSM Agent on a target instance
 directly (no agent   (aws:runCommand — requires the instance to be
 needed — e.g.        a Managed Instance with SSM Agent running)
 ModifyVolume)              │
   │                  │
   ▼                  ▼
Response captured as step output
      │
      ▼
Next step runs, or automation ends / aborts on failure
      │
      ▼
Full execution history stored (viewable in console + CloudTrail)
```

Important distinction: steps using `aws:executeAwsApi`, `aws:changeInstanceState`, and `aws:waitForAwsResourceProperty` talk **directly to the AWS control plane** — they do **not** need the SSM Agent running inside the instance. Only steps that need to run something *inside the OS* (like `aws:runCommand` to extend a filesystem) depend on the SSM Agent and require the instance to already be a **Managed Instance**.

---

## 8. Execution Modes — Single Target vs Rate Control (Fleet-Wide)

- **Single execution** — the runbook runs once, against one specified target (e.g., one Instance ID passed as a parameter). This is what the hands-on lab below builds.
- **Rate control / multi-target execution** — the same runbook is executed against many targets (by tag, resource group, or explicit list), with `MaxConcurrency` and `MaxErrors` controlling how many run in parallel and how many failures are tolerated before the whole automation stops. This is how you'd run the same resize runbook safely across 50 instances instead of one.

---

## 9. Approval Workflows Inside a Runbook

For higher-risk runbooks (e.g., anything that stops production instances), you can insert an `aws:approve` step that pauses execution and sends a notification (via SNS) to a named approver. The automation will not proceed past that step until the approver explicitly approves or rejects it in the console or via API — turning a fully automated runbook into a **human-gated** one for the riskiest steps, without giving up the consistency benefits of automation for everything else.

---

## 10. Hands-On Lab — Build a "Stop → Snapshot → Resize → Start → Verify" Runbook

### Lab Objective

Build and execute a custom Automation runbook that safely resizes an EC2 instance's root EBS volume:

1. Look up the instance's root volume
2. Stop the instance
3. Take a safety snapshot of the root volume
4. Wait for the snapshot to complete
5. Resize the volume
6. Wait for the resize to finish
7. Start the instance back up
8. Wait for the instance to pass status checks
9. (Follow-on) Extend the filesystem/partition inside the OS so the new space is actually usable

### Prerequisites

- One EC2 instance (Linux, e.g. Amazon Linux 2023 or Ubuntu) already registered as an SSM **Managed Instance** (see the SSM core guide, section on requirements — IAM role with `AmazonSSMManagedInstanceCore`, outbound HTTPS reachability)
- The instance's root volume is `gp2` or `gp3` (both support live `ModifyVolume` resizing)
- Permission to create IAM roles and SSM documents in the account

### Lab Architecture

```
You (Console)
     │
     ▼
Systems Manager → Automation → Execute automation
     │
     ▼
Custom Runbook: "SafeVolumeResize"
     │
     ├─▶ Step 1: getInstanceDetails        (aws:executeAwsApi)
     ├─▶ Step 2: getRootVolumeId           (aws:executeAwsApi)
     ├─▶ Step 3: stopInstance              (aws:changeInstanceState)
     ├─▶ Step 4: waitForInstanceStopped    (aws:waitForAwsResourceProperty)
     ├─▶ Step 5: createSnapshot            (aws:executeAwsApi)
     ├─▶ Step 6: waitForSnapshotCompleted  (aws:waitForAwsResourceProperty)
     ├─▶ Step 7: resizeVolume              (aws:executeAwsApi)
     ├─▶ Step 8: waitForResizeCompleted    (aws:waitForAwsResourceProperty)
     ├─▶ Step 9: startInstance             (aws:changeInstanceState)
     └─▶ Step 10: waitForStatusChecksOk    (aws:waitForAwsResourceProperty)
```

### Step-by-Step Sequence (Console-First)

**Step A — Create the IAM role the runbook will assume**

1. Go to **IAM → Roles → Create role**.
2. Trusted entity: **AWS service** → **Systems Manager** (or a custom trust policy trusting `ssm.amazonaws.com`).
3. Attach policy `AmazonSSMAutomationRole` (or a scoped custom policy granting `ec2:DescribeInstances`, `ec2:DescribeVolumes`, `ec2:CreateSnapshot`, `ec2:DescribeSnapshots`, `ec2:ModifyVolume`, `ec2:DescribeVolumesModifications`, `ec2:StartInstances`, `ec2:StopInstances`, `ec2:DescribeInstanceStatus`).
4. Name it `SSM-SafeVolumeResize-Role` and create it.

**Step B — Create the custom Automation document**

1. Go to **Systems Manager → Documents → Create document → Automation**.
2. Name it `SafeVolumeResize`.
3. Switch the editor to **YAML** and paste the full document below (Step C).
4. Click **Create automation**.

**Step C — The full runbook document**

```yaml
description: "Safely resize an EC2 instance's root EBS volume: stop, snapshot, resize, start, verify"
schemaVersion: '0.3'
assumeRole: '{{ AutomationAssumeRole }}'

parameters:
  InstanceId:
    type: String
    description: "(Required) The EC2 instance whose root volume will be resized"
  NewVolumeSizeGiB:
    type: Integer
    description: "(Required) The new size, in GiB, for the root volume"
  AutomationAssumeRole:
    type: String
    description: "(Required) ARN of the IAM role Automation assumes to perform these actions"
    default: ''

mainSteps:
  - name: getInstanceDetails
    action: aws:executeAwsApi
    onFailure: Abort
    inputs:
      Service: ec2
      Api: DescribeInstances
      InstanceIds:
        - '{{ InstanceId }}'
    outputs:
      - Name: rootDeviceName
        Selector: '$.Reservations[0].Instances[0].RootDeviceName'
        Type: String
    nextStep: getRootVolumeId

  - name: getRootVolumeId
    action: aws:executeAwsApi
    onFailure: Abort
    inputs:
      Service: ec2
      Api: DescribeVolumes
      Filters:
        - Name: attachment.instance-id
          Values: ['{{ InstanceId }}']
        - Name: attachment.device
          Values: ['{{ getInstanceDetails.rootDeviceName }}']
    outputs:
      - Name: rootVolumeId
        Selector: '$.Volumes[0].VolumeId'
        Type: String
    nextStep: stopInstance

  - name: stopInstance
    action: aws:changeInstanceState
    onFailure: Abort
    inputs:
      InstanceIds:
        - '{{ InstanceId }}'
      DesiredState: stopped
    nextStep: waitForInstanceStopped

  - name: waitForInstanceStopped
    action: aws:waitForAwsResourceProperty
    timeoutSeconds: 300
    onFailure: Abort
    inputs:
      Service: ec2
      Api: DescribeInstances
      InstanceIds:
        - '{{ InstanceId }}'
      PropertySelector: '$.Reservations[0].Instances[0].State.Name'
      DesiredValues:
        - stopped
    nextStep: createSnapshot

  - name: createSnapshot
    action: aws:executeAwsApi
    onFailure: Abort
    inputs:
      Service: ec2
      Api: CreateSnapshot
      VolumeId: '{{ getRootVolumeId.rootVolumeId }}'
      Description: 'Pre-resize safety snapshot for {{ InstanceId }}'
    outputs:
      - Name: snapshotId
        Selector: '$.SnapshotId'
        Type: String
    nextStep: waitForSnapshotCompleted

  - name: waitForSnapshotCompleted
    action: aws:waitForAwsResourceProperty
    timeoutSeconds: 600
    onFailure: Abort
    inputs:
      Service: ec2
      Api: DescribeSnapshots
      SnapshotIds:
        - '{{ createSnapshot.snapshotId }}'
      PropertySelector: '$.Snapshots[0].State'
      DesiredValues:
        - completed
    nextStep: resizeVolume

  - name: resizeVolume
    action: aws:executeAwsApi
    onFailure: Abort
    inputs:
      Service: ec2
      Api: ModifyVolume
      VolumeId: '{{ getRootVolumeId.rootVolumeId }}'
      Size: '{{ NewVolumeSizeGiB }}'
    nextStep: waitForResizeCompleted

  - name: waitForResizeCompleted
    action: aws:waitForAwsResourceProperty
    timeoutSeconds: 600
    onFailure: Abort
    inputs:
      Service: ec2
      Api: DescribeVolumesModifications
      VolumeIds:
        - '{{ getRootVolumeId.rootVolumeId }}'
      PropertySelector: '$.VolumesModifications[0].ModificationState'
      DesiredValues:
        - completed
        - optimizing
    nextStep: startInstance

  - name: startInstance
    action: aws:changeInstanceState
    onFailure: Abort
    inputs:
      InstanceIds:
        - '{{ InstanceId }}'
      DesiredState: running
    nextStep: waitForStatusChecksOk

  - name: waitForStatusChecksOk
    action: aws:waitForAwsResourceProperty
    timeoutSeconds: 300
    isEnd: true
    inputs:
      Service: ec2
      Api: DescribeInstanceStatus
      InstanceIds:
        - '{{ InstanceId }}'
      PropertySelector: '$.InstanceStatuses[0].InstanceStatus.Status'
      DesiredValues:
        - ok
```

**Step D — Execute the runbook**

1. Go to **Systems Manager → Documents → Owned by me** → select `SafeVolumeResize`.
2. Click **Execute automation**.
3. Fill in the parameters:
   - `InstanceId`: your test instance's ID
   - `NewVolumeSizeGiB`: a size larger than the current volume (e.g., current 8 → new 16)
   - `AutomationAssumeRole`: the ARN of `SSM-SafeVolumeResize-Role` from Step A
4. Click **Execute**.

**Step E — Watch the execution**

1. On the execution page, watch each step turn from **In Progress → Success** in sequence: `getInstanceDetails → getRootVolumeId → stopInstance → waitForInstanceStopped → createSnapshot → waitForSnapshotCompleted → resizeVolume → waitForResizeCompleted → startInstance → waitForStatusChecksOk`.
2. Click any step to view its exact input and output — this is your audit trail.
3. Confirm the final step shows **Success**, meaning the instance passed status checks after the resize.

**Step F — Extend the filesystem (follow-on, required for the new space to be usable)**

`ModifyVolume` resizes the underlying EBS block device, but it does **not** automatically extend the partition or filesystem inside the OS — that is a well-known and important nuance. Once the instance is back up:

1. Connect via **Session Manager** (no SSH needed — see the core SSM guide).
2. For an `ext4`-based Linux root volume:

```
growpart /dev/xvda 1
resize2fs /dev/xvda1
```

3. For an `xfs`-based root volume, use `xfs_growfs /` instead of `resize2fs`.
4. Confirm with `df -h` that the filesystem now reflects the new size.

This step can itself be added as an eleventh `aws:runCommand` step in the runbook (invoking `AWS-RunShellScript` with the `growpart`/`resize2fs` commands) to make the entire process — including the OS-level extension — fully automated end to end.

---

## 11. Real-World Scenarios and Use Cases

### Scenario 1: Emergency disk-full remediation
A CloudWatch alarm fires when root volume disk usage crosses 90%. An EventBridge rule triggers the `SafeVolumeResize` runbook automatically with a pre-set `NewVolumeSizeGiB`, resizing the volume before the disk actually fills up and the application crashes — without waking anyone up.

### Scenario 2: Patch rollback runbook
A runbook that stops an instance, restores its root volume from the pre-patch snapshot (taken automatically before every patch run), and starts it back up — a one-click rollback path when a patch causes an unexpected regression.

### Scenario 3: Approval-gated production changes
The same resize runbook, but with an `aws:approve` step inserted right before `stopInstance`, so a senior engineer must approve via SNS notification before any production instance is actually stopped — automation for consistency, human judgment for risk.

### Scenario 4: Fleet-wide chaos engineering
A runbook using `aws:changeInstanceState` to stop and restart instances tagged `ChaosEligible=true` on a schedule, verifying the application's self-healing and failover behavior actually works — deliberately, in a controlled window, not by surprise.

### Scenario 5: Cost optimization scheduling
A runbook scheduled via Maintenance Windows to stop all instances tagged `Environment=Dev` every night at 8 PM and start them again at 8 AM, cutting non-production compute costs without anyone needing to remember to do it manually.

---

## 12. Edge Cases and Failure Scenarios

| Scenario | Root Cause | Resolution |
|---|---|---|
| `stopInstance` step fails immediately | The `AutomationAssumeRole` lacks `ec2:StopInstances` permission | Add the missing action to the role's policy and re-run |
| `waitForSnapshotCompleted` times out | Snapshot is unusually large, or `timeoutSeconds` is too low for the volume size | Increase `timeoutSeconds` proportionally to expected volume size |
| `resizeVolume` fails with a size error | `NewVolumeSizeGiB` is smaller than or equal to the current size — EBS only supports growing, not shrinking, via `ModifyVolume` | Pass a value strictly larger than the current volume size |
| `waitForResizeCompleted` never reaches `completed` | Some volume types report `optimizing` for an extended period after resize, which is expected, not a failure | Accept both `completed` and `optimizing` as desired values (already handled in the lab document) |
| Instance starts but never reaches status `ok` | Underlying hardware issue, or the resize corrupted something at the OS level (rare) | Investigate via Session Manager; worst case, restore from the safety snapshot taken in Step 5 |
| Filesystem still shows old size after the runbook succeeds | Expected — `ModifyVolume` doesn't extend the filesystem automatically | Run Step F manually, or add the `aws:runCommand` follow-on step |
| Runbook execution stuck at an `aws:approve` step forever | No one has approved or rejected it yet | Have the named approver act on it, or cancel the execution and adjust the approver list |

---

## 13. Troubleshooting Table

| Symptom | Check | Fix |
|---|---|---|
| Execution shows "Failed" at `getRootVolumeId` | Root device name selector didn't match any volume | Verify the instance actually has an attached root volume with a matching `attachment.device` value |
| Execution shows "Failed" at `stopInstance`/`startInstance` | IAM role permissions | Confirm `AutomationAssumeRole` has both `ec2:StopInstances` and `ec2:StartInstances` |
| `aws:executeAwsApi` step fails with `AccessDenied` | The Automation role, not your own console user's permissions, is what's used at runtime | Update the role attached in the `AutomationAssumeRole` parameter, not your personal IAM user |
| Steps run in the wrong order | Missing or incorrect `nextStep` values | Double-check every step's `nextStep` points to the exact `name` of the intended next step |
| Automation halts with no error after one step | That step's `isEnd: true` was set prematurely | Remove `isEnd: true` from any step that isn't meant to be the final one |

---

## 14. Security Considerations

- The `AutomationAssumeRole` should be scoped to only the specific EC2 actions the runbook needs — avoid attaching broad managed policies like `AmazonEC2FullAccess` to an automation role.
- Every execution is recorded in CloudTrail as calls made by the assumed automation role, giving a clear audit trail distinct from any individual user's actions.
- For runbooks that act on production resources, prefer inserting an `aws:approve` step over granting broad "just run it" access to more people.
- Store any parameter values that are secrets (not needed in this particular lab, but common in others) in Parameter Store `SecureString` parameters and reference them, rather than passing secrets as plain automation parameters.

---

## 15. Interview Questions

**Q1. What is an SSM Automation runbook?**
A YAML/JSON document defining an ordered sequence of steps — calling AWS APIs, running scripts, waiting on conditions, or requesting approval — that Systems Manager executes automatically and consistently, with a full audit trail.

**Q2. What's the difference between `aws:executeAwsApi` and `aws:runCommand`?**
`aws:executeAwsApi` calls the AWS control plane directly and does not need the SSM Agent. `aws:runCommand` executes inside the operating system via the SSM Agent, so the target must already be a Managed Instance.

**Q3. Why does the lab runbook take a snapshot before resizing the volume?**
As a safety rollback point — if anything goes wrong after the resize (corruption, instance failing status checks), the original state can be restored from the snapshot.

**Q4. Why doesn't `ModifyVolume` alone make the new space usable?**
`ModifyVolume` only changes the size of the underlying EBS block device. The partition table and filesystem inside the OS must still be extended separately (`growpart`/`resize2fs` or `xfs_growfs` on Linux, Disk Management on Windows).

**Q5. What does the `assumeRole` field control, and why does it matter?**
It defines which IAM role Automation uses to make its API calls — meaning the runbook's permissions are independent of whoever triggers the execution, which is central to both security and consistency.

**Q6. How would you make a risky runbook require human sign-off?**
Insert an `aws:approve` step before the risky action; the automation pauses and notifies a named approver via SNS, and won't proceed until approved.

**Q7. How can a runbook be triggered automatically instead of manually?**
Via an EventBridge rule reacting to a CloudWatch alarm or scheduled cron expression, or via a Maintenance Window task.

**Q8. What happens if a step fails and `onFailure` is not explicitly set?**
The default behavior is to abort the entire automation execution at that step.

---

## 16. Cheat Sheet

```
Define the whole document                → schemaVersion: '0.3'
Who the runbook acts as                   → assumeRole: '{{ AutomationAssumeRole }}'
Call an AWS API directly                  → aws:executeAwsApi
Start/stop/reboot an instance             → aws:changeInstanceState
Poll until a condition is true            → aws:waitForAwsResourceProperty
Run something inside the OS               → aws:runCommand (needs SSM Agent / Managed Instance)
Pause for a human decision                → aws:approve
Conditional logic                          → aws:branch
Reference an earlier step's output        → {{ stepName.outputName }}
Control what runs next                     → nextStep
Stop everything on failure                 → onFailure: Abort
Mark the final step                        → isEnd: true
Run against many instances at once         → Rate control (MaxConcurrency / MaxErrors)
```

---

## 17. Mastery Checklist

- [ ] Can explain the difference between an AWS-owned and a custom runbook
- [ ] Can describe every top-level field in an Automation document (`schemaVersion`, `assumeRole`, `parameters`, `mainSteps`)
- [ ] Can explain the difference between `aws:executeAwsApi` and `aws:runCommand`
- [ ] Can trace the full step sequence of the Stop → Snapshot → Resize → Start → Verify runbook from memory
- [ ] Can explain why a snapshot happens before the resize, not after
- [ ] Can explain why the filesystem still needs manual extension after `ModifyVolume`
- [ ] Can add an `aws:approve` step to gate a risky action
- [ ] Can describe how a runbook gets triggered automatically via EventBridge
- [ ] Can diagnose a failed step using its recorded input/output in the execution history
- [ ] Has personally executed this lab against a real test instance and watched every step succeed

---

## 18. Cleanup Steps

Perform in this order to avoid leaving orphaned resources:

1. If the lab instance is still stopped/running only for this test, terminate it (or leave it as-is if it's a shared test instance).
2. Delete the safety snapshot created during the lab, once you've confirmed it's no longer needed as a rollback point.
3. Delete the custom Automation document `SafeVolumeResize` if it was created purely for this lab (Systems Manager → Documents → Owned by me → Delete).
4. Detach and delete the `SSM-SafeVolumeResize-Role` IAM role if it isn't reused elsewhere.
5. Review the automation's execution history entry — it remains visible for auditing even after the document is deleted, which is expected and generally fine to leave.

---

## Conclusion

SSM Automation runbooks turn a team's tribal, manually-executed operational knowledge into a versioned, auditable, and repeatable artifact — the same category of value Terraform brought to infrastructure provisioning, applied instead to operational procedures. The lab in this guide is a realistic, production-shaped example: it doesn't just demonstrate the syntax, it encodes an actual safety practice (snapshot before resize) that's easy to skip when done manually under pressure, and impossible to skip once it's baked into the runbook itself.
