# AWS Config: Complete Study Guide (Verified Version)

> Scope: concepts only. No CLI, no Terraform.
> This version was checked against AWS documentation. Limits, resource-type lists and prices change often, so confirm exact numbers in the current AWS docs before using them in production.

---

## Table of Contents

1. [What AWS Config is](#1-what-aws-config-is)
2. [Why it exists](#2-why-it-exists)
3. [AWS Config vs CloudTrail vs CloudWatch](#3-aws-config-vs-cloudtrail-vs-cloudwatch)
4. [Configuration vs configuration history](#4-configuration-vs-configuration-history)
5. [Resource types supported](#5-resource-types-supported)
6. [Regional nature of Config](#6-regional-nature-of-config)
7. [What Config does not monitor](#7-what-config-does-not-monitor)
8. [Configuration snapshots](#8-configuration-snapshots)
9. [How Config collects and stores data](#9-how-config-collects-and-stores-data)
10. [Configuration Items (CI)](#10-configuration-items-ci)
11. [Config Rules](#11-config-rules)
12. [Remediation](#12-remediation)
13. [Related features and cost](#13-related-features-and-cost)
14. [Quick revision cheat sheet](#14-quick-revision-cheat-sheet)

---

## 1. What AWS Config is

AWS Config is a **resource inventory + change-tracking + compliance** service.

It does three jobs:

1. **Records** the configuration of your supported AWS resources (what settings they have).
2. **Tracks changes** to those settings over time (a timeline per resource).
3. **Evaluates** those settings against rules you define and tells you whether each resource is **compliant or non-compliant**.

Example with an S3 bucket:
- Config records: "Bucket X has versioning off and default encryption off."
- Later someone turns versioning on. Config records a new version of that configuration.
- A rule says "buckets must have encryption on." Config marks Bucket X **NON_COMPLIANT**.

**Important mindset:** Config is **detective**, not preventive. It tells you something is wrong *after* the change happened. It does not block the change. The most it can do is fix it afterwards through remediation.

---

## 2. Why it exists

| Need | How Config helps |
|---|---|
| "What does my environment look like right now?" | Resource inventory |
| "What did it look like last Tuesday?" | Configuration history |
| "What changed, and what else is connected to it?" | Change timeline + relationships |
| Compliance and audit (CIS, PCI-DSS, HIPAA, internal policy) | Rules, compliance dashboard, evidence |
| Security analysis | Detect open security groups, unencrypted volumes, public buckets, etc. |
| Troubleshooting ("it worked yesterday") | Compare configuration before and after |
| Fix drift automatically | Remediation using SSM Automation |

---

## 3. AWS Config vs CloudTrail vs CloudWatch

These answer **different questions**.

| | **AWS Config** | **AWS CloudTrail** | **Amazon CloudWatch** |
|---|---|---|---|
| **Core question** | *What does the resource look like, and is it compliant?* | *Who made which API call, and when?* | *How is it performing right now?* |
| **Records** | Resource **configuration state** | **API activity** (console, CLI, SDK, API) | **Metrics, logs, alarms** |
| **Example** | "Security group sg-123 allows 0.0.0.0/0 on port 22" | "User `ravi` called `AuthorizeSecurityGroupIngress` at 10:42" | "EC2 CPU has been 92% for 10 minutes" |
| **Compliance checks** | Yes (Config Rules) | No | No |
| **Typical use** | Audit, drift, change history | Security investigation, who-did-it | Monitoring, alerting, dashboards |

### How they work together

Incident: *SSH port 22 was opened to the world.*

- **Config** shows the security group is non-compliant and shows the before/after configuration.
- **CloudTrail** shows which user or role made the API call.
- **CloudWatch** (with EventBridge/alarms) can alert you and holds logs and metrics.

**How Config connects to CloudTrail:** in the Config console, the **Resource Timeline** can be filtered by *Configuration events*, *Compliance events* and *CloudTrail events*. For many changes the timeline shows a link to the matching CloudTrail event. Config finds it by looking up CloudTrail events, and this only works for events from roughly the **last 90 days**. This link is a console convenience. It is **not** stored inside the configuration item (see section 10).

> Rule of thumb: **Config = WHAT (state), CloudTrail = WHO (action), CloudWatch = HOW WELL (performance).**

---

## 4. Configuration vs configuration history

| Term | Meaning |
|---|---|
| **Configuration** | The **current** settings of a resource (latest recorded state). |
| **Configuration history** | The **series of past configurations** of a resource. Each recorded change adds one configuration item (CI) to the series. |

Example, an EC2 instance:
- Configuration (now): `t3.large`, security group `sg-A`.
- Configuration history: created as `t3.micro` -> changed to `t3.medium` -> changed to `t3.large` -> security group changed from `sg-B` to `sg-A`.

Where you see history:
- **Resource Timeline** in the Config console (changes as CIs over time).
- **S3 bucket**: Config delivers *configuration history files* (see sections 8 and 9).
- **Compliance history**: Config can also keep the history of a resource's *compliance* results over time. This is recorded as a special resource type, `AWS::Config::ResourceCompliance`. Recording all resource types includes it. If you record only specific types, you must include it to see compliance history on the timeline.

---

## 5. Resource types supported

Config supports a **large and growing list** of AWS resource types across most major services. Examples:

- **Compute:** EC2 instances, Auto Scaling groups, Lambda functions
- **Networking:** VPCs, subnets, route tables, security groups, internet gateways, NAT gateways, transit gateways, load balancers
- **Storage:** S3 buckets, EBS volumes
- **Database:** RDS instances, DynamoDB tables
- **Security/Identity:** IAM users, roles, groups, policies; KMS keys
- **Others:** CloudFormation stacks, CloudTrail trails, ECS, EKS, SQS, SNS, Secrets Manager, and many more

Not every resource type is available in every region. Check the current "resource coverage by region" list.

### Recording options

**Recording strategy** (what to record):
- **All current and future supported resource types** in the region (default and simplest). You can still override the frequency for some types or exclude some types.
- **Specific resource types only.**

**Recording frequency** (how often):
- **Continuous**: a CI is created whenever a change is detected.
- **Daily**: at most one CI per day, reflecting the latest state. Cheaper, but you lose the in-between changes. **Daily recording is not supported for every resource type.**

**Global resources:**
- **IAM resource types** (users, groups, roles, policies) are global, and they are **initially excluded from recording** in the setup console to help reduce cost. You must turn them on deliberately.
- Rules differ depending on when a global resource type was added to Config (before or after February 2022), and where it can be recorded. See section 6.

**Third-party and custom resources:** Config also lets you record resources that are not native AWS resources (for example on-prem servers or SaaS resources) by registering them as custom resource types.

> A resource type must be **recorded** for Config to track it and for rules to have data to evaluate.

---

## 6. Regional nature of Config

**AWS Config is regional.** This catches many beginners.

- Config must be **turned on separately in every region** you want covered.
- Each region has **its own configuration recorder, delivery channel, rules and data**.
- You can have **only one delivery channel per region per account**.
- Enabling Config in `ap-south-1` does **not** record resources in `us-east-1`.

### Global resources

Global resources are not tied to one region (the ARN has no region).

- **Global IAM types** (users, groups, roles, policies): these can only be recorded in the regions where Config was available before February 2022. To avoid **duplicate CIs and duplicate cost**, record them **in one region only**.
- **Global types added to Config after February 2022:** recorded only in the service's **home region**.
- **Aurora global clusters:** a special case. If you enable recording in one region, Config records CIs for it in **all** your enabled regions.

(Always check the current AWS page "Recording AWS resources > Global resources" for the exact list of regions and types.)

### Seeing everything in one place: Aggregators

An **aggregator** collects Config data (resource inventory and compliance) from **multiple regions and multiple accounts** into one view.
- It is a **read-only view**. It does not turn Config on for you. Config must still be enabled in each source account and region.
- Works with accounts in an **AWS Organization** or individually authorized accounts.

---

## 7. What Config does not monitor

Config is **not** a catch-all monitoring tool.

| Config does NOT do this | Use this instead |
|---|---|
| Show **who** made an API call | **CloudTrail** |
| Performance metrics (CPU, memory, latency) | **CloudWatch** |
| Application and OS logs | **CloudWatch Logs** |
| Network traffic | **VPC Flow Logs** |
| Threat detection | **GuardDuty** |
| Contents of your data (S3 objects, database rows) | Not recorded. Config records **settings**, not **data** |
| Resource types it does not support | Not tracked, unless you add them as custom resource types |
| Resource types you excluded, or regions where the recorder is off | Not tracked |
| Changes while the recorder is stopped | Not recorded, so you get a gap in history |
| **Block or prevent** a bad change | IAM policies, SCPs, CloudFormation Hooks (preventive tools) |
| Instant detection | Config detects **after** the change is recorded, so there is a delay |

Note: by default Config does not look inside an instance's operating system. The exception is the optional Systems Manager (SSM) managed-instance inventory resource type, which brings some software inventory data into Config.

---

## 8. Configuration snapshots

A **configuration snapshot** is a **point-in-time picture of all resources Config records**, saved as a **JSON file in S3**.

- Think: "take a photo of my whole environment right now."
- **Not automatic by default.** Turning on the recorder and delivery channel does *not* by itself produce snapshots. You get them in two ways:
  - **On demand**: you request a snapshot (the `DeliverConfigSnapshot` action).
  - **On a schedule**: you set a snapshot **delivery frequency** on the delivery channel: **every 1, 3, 6, 12 or 24 hours**.
- Delivered to the **S3 bucket** of your delivery channel. Config can also send **SNS notifications** about delivery (started, completed, failed).

### Snapshot vs history file

| | Configuration snapshot | Configuration history file |
|---|---|---|
| Scope | **All** recorded resources at one moment | Resources of **one resource type** that **changed** |
| Frequency | On demand, or scheduled (1/3/6/12/24 h) | Every **6 hours** |
| If nothing changed | A snapshot still contains the full state | **No file is sent** |
| Good for | Audits, baselines, "state of everything at time X" | "How did this resource change?" |

> Both files in S3 are for your **auditing and analysis**. Config keeps its own copy of CIs for the console timeline and queries.

---

## 9. How Config collects and stores data

### Building blocks

| Component | What it does |
|---|---|
| **Configuration recorder** | The engine. Detects changes to supported resources and records them as CIs. Must be **started**. |
| **IAM role** | Gives Config permission to read your resources and write to S3 and SNS. |
| **Delivery channel** | Defines **where** Config delivers data: an **S3 bucket** (required) and an **SNS topic** (optional). One per region per account. |
| **Config's own store** | Holds recorded CIs so the console timeline, queries and rules work. |
| **S3 bucket** | Long-term storage of history files and snapshots. |

### The flow

```
A resource changes in your account
        |
        v
Configuration recorder detects the change
        |
        v
Config collects the resource's configuration and creates a Configuration Item (CI)
        |
        +--> Stored in Config (timeline, queries, rule evaluation)
        +--> Included in history files delivered to S3 (every 6 hours)
        +--> Included in snapshots (on demand or scheduled)
        +--> Notification via SNS / EventBridge (optional)
        +--> Change-triggered Config Rules evaluate the resource
```

Useful details:
- When you first turn Config on, it **discovers existing resources** and creates a CI for each one.
- Config **records the state that resulted from a change**. If several changes happen to a resource in quick succession, you may see fewer CIs than individual API calls.
- Config also **periodically scans** resources for changes it has not yet recorded, including changes that did not come through that resource's own API calls.

### Retention

You can set how long Config keeps **configuration item history**: from **30 days to 2557 days (about 7 years)**. This setting currently applies **only to CI history**. There is **one retention configuration per region per account**.

### Notifications

Through **SNS** or **EventBridge**, Config can notify you about:
- Configuration changes (each new CI)
- Compliance changes (for example compliant -> non-compliant)
- Snapshot and history delivery status

These let you trigger alerts, tickets or automation.

---

## 10. Configuration Items (CI)

A **Configuration Item (CI)** is the **basic unit of data in Config**: a JSON record describing **one resource at one point in time**.

> Each time a resource's configuration is recorded as changed, Config creates a **new CI**. Configuration history is simply the list of CIs for that resource.

### 10.1 What a CI actually contains

| Field | Meaning |
|---|---|
| `version` | Version of the CI format |
| `accountId` | Account that owns the resource |
| `resourceId` | Unique ID (for example `i-0abc123`, `sg-0123`, a bucket name) |
| `resourceType` | Type, such as `AWS::EC2::Instance` |
| `resourceName` | Name, if the resource has one |
| `arn` | The resource's ARN |
| `awsRegion`, `availabilityZone` | Where it lives |
| `resourceCreationTime` | When the resource was created |
| `tags` | Tags on the resource |
| `configuration` | The **main settings** of the resource |
| `supplementaryConfiguration` | Extra attributes for certain resource types |
| `relationships` | Links to other related resources |
| `relatedEvents` | **Empty in current CI versions (1.3 and later).** Do not rely on it for CloudTrail links. |
| `configurationItemCaptureTime` | **When Config captured** this CI |
| `configurationItemDeliveryTime` | When the CI was delivered |
| `configurationItemStatus` | Status of this CI (see 10.3) |
| `configurationStateId` | A number that orders CIs (later = higher) |
| `configurationItemMD5Hash` | Hash used to detect whether the configuration changed |
| `recordingFrequency` | Whether it was recorded continuously or daily |

### 10.2 The key fields in more detail

**Resource ID**
- The identifier of the resource. Together with the resource type (and region), it identifies what the CI is about.
- Example: `sg-0a1b2c3d4e5f`.

**Resource type**
- Format: `AWS::<Service>::<Resource>`. Examples: `AWS::EC2::Volume`, `AWS::IAM::Role`, `AWS::Lambda::Function`.
- Rules are scoped by resource type.

**Configuration**
- The actual settings as JSON. For a security group: its rules, VPC ID, description. For an EBS volume: size, type, encrypted yes/no, attachments.
- This is what **rules evaluate**.

**Relationships**
- Shows how the resource connects to others.
- Example: an EC2 instance's CI lists relationships to its **security groups, subnet, VPC, EBS volumes and network interfaces**.
- Relationship names look like "Is associated with ...", "Is contained in ...", "Is attached to ...".
- Why it matters: **impact analysis**. If you change a security group, you can see which instances use it.
- **One change can produce several CIs.** When a resource that has relationships changes (for example an instance gets a different security group), Config can create CIs for the related resources too, because their relationships changed.

**Capture time**
- `configurationItemCaptureTime` = when Config recorded this state. It is the timestamp shown on the timeline.
- It is when Config **recorded** the change, which can be slightly after the change happened.

**Previous configuration**
- A CI does **not** carry a "previous configuration" copy of itself. The previous configuration is the **earlier CI** in the history.
- Ways to see what changed:
  - **Console timeline:** shows the changes between points in time.
  - **Change notifications (SNS):** the message includes a `configurationItemDiff` section. It lists `changedProperties` with `previousValue`, `updatedValue` and a `changeType` of `CREATE`, `UPDATE` or `DELETE`. For example a volume's state changing from `available` (previousValue) to `in-use` (updatedValue).

### 10.3 CI status (`configurationItemStatus`)

| Status | Meaning |
|---|---|
| `OK` | Resource exists and the CI was recorded normally |
| `ResourceDiscovered` | Config found the resource for the first time (newly discovered) |
| `ResourceNotRecorded` | Resource was discovered, but its configuration is not recorded because the recorder does not record this resource type |
| `ResourceDeleted` | The resource was deleted, and the deletion was recorded |
| `ResourceDeletedNotRecorded` | The resource was deleted, but this type is not being recorded |

### 10.4 Example CI (simplified)

```json
{
  "configurationItemCaptureTime": "2026-10-06T08:15:30.000Z",
  "configurationItemStatus": "OK",
  "resourceType": "AWS::EC2::SecurityGroup",
  "resourceId": "sg-0a1b2c3d4e5f",
  "awsRegion": "ap-south-1",
  "tags": { "Env": "prod" },
  "relationships": [
    { "resourceType": "AWS::EC2::VPC", "resourceId": "vpc-01234", "relationshipName": "Is contained in Vpc" }
  ],
  "configuration": {
    "groupName": "web-sg",
    "ipPermissions": [
      { "fromPort": 22, "toPort": 22, "ipRanges": ["0.0.0.0/0"] }
    ]
  }
}
```

A rule like "no SSH open to the world" reads `configuration` and marks this resource **NON_COMPLIANT**.
(This is a simplified teaching example. Real CIs have more fields and a slightly different layout.)

### 10.5 Oversized CIs

If a change notification would exceed the maximum size Amazon SNS allows, Config sends an **oversized** notification instead. Custom Lambda rules must handle this case by fetching the full CI themselves (see 11.4).

---

## 11. Config Rules

### 11.1 What Config Rules are

A **Config Rule** defines the **desired configuration** of resources. Config evaluates in-scope resources against it and gives each one a result.

**Compliance results:**

| Result | Meaning |
|---|---|
| `COMPLIANT` | Resource meets the rule |
| `NON_COMPLIANT` | Resource breaks the rule |
| `NOT_APPLICABLE` | Rule does not apply to this resource |
| `INSUFFICIENT_DATA` | Not enough information to decide. **Config uses this itself. A custom rule cannot report it.** |

The **rule itself** is shown as non-compliant if **any** resource in its scope is non-compliant.

**Rule scope** (what the rule checks) can be limited by:
- **Resource type** (for example only S3 buckets)
- **Tag** key/value (for example only `Env=prod`)
- **A specific resource ID**

**Evaluation modes:**
- **Detective:** checks resources that already exist (the classic mode).
- **Proactive:** checks a resource's planned configuration **before it is deployed** (for example from a deployment pipeline). **Not all rules support proactive mode.**

### 11.2 AWS-managed rules

- **Predefined rules written and maintained by AWS.** You choose one, set its parameters and turn it on. No code.
- They cover common best practices.
- Examples:
  - `s3-bucket-public-read-prohibited`: no public-read S3 buckets
  - `encrypted-volumes`: EBS volumes must be encrypted
  - `restricted-ssh`: no unrestricted incoming SSH
  - `required-tags`: resources must have specific tags
  - `root-account-mfa-enabled`: root user must have MFA
  - `iam-password-policy`: password policy meets your requirements
  - `rds-instance-public-access-check`: RDS instances must not be public
- Many accept **parameters** (for example the tag keys that are required).
- Not every managed rule is available in every region.
- Best starting point. Use these before writing anything custom.

### 11.3 Custom rules

When no managed rule fits, you write your own logic. There are two ways:

| Type | How it works |
|---|---|
| **Custom Lambda rule** | You write a **Lambda function**. Most flexible. Supports change-triggered and periodic evaluation. |
| **Custom policy rule (Guard)** | You write the logic in **AWS CloudFormation Guard** (policy-as-code). **No Lambda to manage.** For detective evaluation, these rules are **triggered by configuration changes** (not by a schedule). |

### 11.4 Lambda-based custom rules (how they work)

```
Config decides to evaluate (trigger)
        |
        v
Invokes your Lambda with an event
        |
        v
Lambda runs your logic and decides COMPLIANT / NON_COMPLIANT / NOT_APPLICABLE
        |
        v
Lambda calls PutEvaluations (with the result token) to report back to Config
        |
        v
Config stores the result and updates compliance
```

**What the event contains:**

| Field | Meaning |
|---|---|
| `invokingEvent` | JSON string with the details that triggered the evaluation (the CI for change-triggered rules, or schedule info for periodic rules) |
| `ruleParameters` | JSON string with the key/value parameters you defined for the rule |
| `resultToken` | Token the function **must pass back** when it calls `PutEvaluations` |
| `eventLeftScope` | `true` if the resource is no longer in the rule's scope. The function should not evaluate it normally |
| `executionRoleArn` | The IAM role Config uses for the rule |
| `configRuleArn`, `configRuleName`, `configRuleId` | Identify the rule |
| `accountId`, `version` | Account that owns the rule, and event format version |

**Key points:**
- Your function must report with `PutEvaluations` using the `resultToken`. Otherwise Config does not know which evaluation the result belongs to.
- **Allowed results from the function:** `COMPLIANT`, `NON_COMPLIANT`, `NOT_APPLICABLE`. It **cannot** send `INSUFFICIENT_DATA`.
- **Annotation:** you can add a short explanation of *why* (maximum **256 characters**). Very helpful in audits.
- **Permissions:**
  - Config must be allowed to **invoke** your Lambda.
  - The Lambda's execution role must be allowed to call `config:PutEvaluations`.
- **Message types your function should handle** (`messageType` inside `invokingEvent`):
  - `ConfigurationItemChangeNotification`: a normal change-triggered event.
  - `OversizedConfigurationItemChangeNotification`: the CI was too large for the notification, so the function must retrieve the full CI itself.
  - `ScheduledNotification`: a periodic evaluation.
- Typical checks inside the function: only evaluate if the CI status is `OK` or `ResourceDiscovered`, and `eventLeftScope` is `false`. Return `NOT_APPLICABLE` otherwise.
- If the function fails or never reports a result, the resource will not get a proper result. Test and log well.

When to pick Lambda: complex logic, calls to other systems, or things Guard cannot express.

### 11.5 Periodic evaluation

- The rule runs on a **schedule**: every **1, 3, 6, 12 or 24 hours** (`One_Hour` ... `TwentyFour_Hours`). If you do not choose, the default is **every 24 hours**.
- Evaluates **all in-scope resources** each time, whether or not they changed.
- **Use when:**
  - The check is not tied to one resource changing (for example account-level settings).
  - You want regular re-checking regardless of changes.
- **Downside:** problems can go unnoticed until the next scheduled run.

### 11.6 Change-triggered evaluation

- The rule runs **when Config records a configuration change** for a resource in scope.
- **Use when** the check depends on the resource's own settings (encryption, public access, security group rules).
- **Advantages:** reacts quickly and does not re-check unchanged resources.
- It is **not instant**: Config evaluates only **after** the change has been completed **and recorded**.
- Rule scope (types, tags, resource ID) decides which changes trigger it.

### 11.7 Comparison

| | Change-triggered | Periodic |
|---|---|---|
| Runs when | A resource in scope changes (CI recorded) | Every 1/3/6/12/24 h (default 24 h) |
| Checks | The changed resource | All in-scope resources |
| Detection speed | Fast (after recording) | Up to the interval |
| Best for | Resource-level settings | Account-level or time-based checks |

### 11.8 Other useful facts about rules

- When you **create** a rule, Config evaluates existing resources in scope.
- You can **re-evaluate** a rule manually. This is useful after fixing something, especially for periodic rules, so you do not wait for the next schedule.
- The resource type must be **recorded**, otherwise there is nothing to evaluate.
- Rules are **regional**.
- **Conformance packs** bundle many rules (and remediation actions) into one deployable set, useful for standards such as CIS or PCI.
- **Organization rules** deploy a rule across all accounts in an AWS Organization.
- You pay **per rule evaluation**, so scope rules carefully.

---

## 12. Remediation

This is where Config becomes much more useful operationally. Rules **find** problems, and remediation **fixes** them.

### 12.1 How remediation works

- Remediation is **attached to a Config Rule**.
- The remediation action is an **AWS Systems Manager (SSM) Automation document** (runbook). This is the only target type Config remediation uses.
- So: **Config detects -> SSM Automation fixes.**

```
Rule marks resource NON_COMPLIANT
        |
        v
Remediation triggered (automatic or manual)
        |
        v
Config starts the SSM Automation runbook, passing the resource ID and parameters
        |
        v
Runbook changes the resource (for example, enables encryption)
        |
        v
Resource is re-evaluated -> COMPLIANT (hopefully)
```

### 12.2 Automatic remediation

- Config **runs the fix automatically** for resources evaluated as non-compliant.
- No human involved.
- **Important:** when you **turn on** automatic remediation for a rule, Config starts remediation for **all resources that are already non-compliant** for that rule, not only for future ones. Plan for this before enabling it.
- **Good for:** low-risk, well-understood, reversible fixes.
- **Risk:** a wrong or too-broad fix can break many resources quickly.
- Automatic remediation **requires** an `AutomationAssumeRole` (see 12.5).

### 12.3 Manual remediation

- Config does **nothing by itself**. A person looks at non-compliant resources and **chooses to run the fix**.
- **Good for:** high-impact changes, production resources, or when a human must review first.
- Slower, but safer.

| | Automatic | Manual |
|---|---|---|
| Who triggers | Config | A person |
| Speed | Fast | Slow |
| Risk | Higher | Lower |
| Best for | Safe, repetitive fixes | Risky or production fixes |

### 12.4 SSM Automation

- The engine that actually **performs the fix**.
- A **runbook (Automation document)** is a series of steps (call an AWS API, run a script, wait, approve, and so on).
- You can use:
  - **AWS-provided runbooks**, for example `AWS-EnableS3BucketEncryption` and `AWS-DisableS3BucketPublicReadWrite`. There is also a family of runbooks with names starting `AWSConfigRemediation-`, built for Config remediation.
  - **Your own custom runbooks** for company-specific fixes.
- Runbooks can include **approval steps** (wait for a person to approve before changing anything).
- **Permissions:** the runbook runs using an **IAM role** (the `AutomationAssumeRole`) that SSM can assume. Give it **only** the permissions the fix needs.

### 12.5 Remediation parameters

Parameters are the **inputs** passed to the runbook.

- **Resource ID parameter (dynamic):** you pick which runbook parameter receives the **ID of the non-compliant resource**. Config fills it in for each resource.
- **Static parameters:** fixed values you set (for example a KMS key ID or a tag value).
- **AutomationAssumeRole:** the IAM role the runbook uses. Required for automatic remediation. Manual remediation also needs a role, though it has a little more flexibility.
- Parameter names must **match the runbook's parameter names exactly**. A mismatch is a common cause of failure.

### 12.6 Retry behavior and execution controls (automatic remediation)

You control how persistent Config is, and how fast it runs fixes:

| Setting | Meaning | Default |
|---|---|---|
| **Maximum automatic attempts** | The maximum number of **failed** attempts for auto-remediation | **5** |
| **Retry attempt seconds** (time window) | The time window used to decide whether to stop retrying and add a remediation exception | **60 seconds** |
| **Concurrent execution rate (%)** | Maximum percentage of remediation actions allowed to run in parallel | **10** |
| **Error percentage** | Percentage of errors allowed before SSM stops running more automations for that rule | **50%** |

**How the retry settings work together** (AWS example): with 5 attempts and 50 seconds, if the fix fails **5 times within 50 seconds**, Config **adds a remediation exception** to that resource. Config then stops automatic remediation for it, which prevents endless remediation loops.

Why this matters:
- Retries help with temporary problems (throttling, short delays).
- The concurrency and error settings stop a bad fix from rolling out across hundreds of resources at once.

### 12.7 Remediation failure

When a remediation fails, the resource usually **stays NON_COMPLIANT**. If it keeps failing and the attempt limit is reached within the time window, Config adds a **remediation exception** to the resource, and automatic remediation stops for it.

**Common causes**

| Cause | Example |
|---|---|
| **Permissions** | `AutomationAssumeRole` is missing a permission the fix needs |
| **Wrong parameters** | Parameter name mismatch or invalid value |
| **Resource state** | Resource cannot be changed right now |
| **Missing dependency** | A KMS key, role or bucket the fix needs does not exist |
| **Throttling / limits** | API rate limits reached |
| **Runbook error** | Bug or unhandled case in a custom runbook |
| **Fix does not satisfy the rule** | Runbook succeeds, but the resource still fails the rule |

**How to investigate**
1. Check the **remediation execution status** in Config.
2. Open the matching **SSM Automation execution** and look at **which step failed** and its error.
3. Fix the cause (permissions, parameters, runbook).
4. If a remediation exception was added, remove it if you want automatic remediation to try again. Then **re-run** the remediation or **re-evaluate** the rule.

**Set up alerting** (EventBridge/SNS) so failed fixes do not sit unnoticed.

**Remediation exceptions** are also something you can add yourself, to **exclude specific resources** from automatic remediation (for example a bucket that must stay public on purpose).

### 12.8 Safe remediation design

Automatic fixes change real infrastructure. Design them carefully.

1. **Start with detection only.** Run rules and review results first. Add remediation only after you trust the rule.
2. **Start manual, then automate** the proven, low-risk fixes.
3. **Remember the "enable" effect.** Turning on automatic remediation hits **every existing non-compliant resource** for that rule. Check the list first, and add exceptions for resources that must not be touched.
4. **Test in non-production first.**
5. **Least privilege.** The `AutomationAssumeRole` should do **only** what the fix needs.
6. **Limit scope.** Use tags, resource types or specific accounts so a fix cannot hit everything.
7. **Prefer reversible, non-destructive actions.** Enabling encryption or blocking public access is safer than deleting or terminating things.
8. **Add approval steps** in the runbook for riskier actions.
9. **Tune the controls.** Keep retries limited. Keep the concurrency percentage and error percentage low for risky fixes.
10. **Watch for fights with your deployment tooling.** If your pipeline keeps re-creating the "bad" setting and Config keeps fixing it, you get a loop. Fix the **source** (the template or pipeline) as well.
11. **Use exceptions** for resources with a valid reason to differ.
12. **Make runbooks idempotent**, so running them twice causes no harm.
13. **Monitor and alert** on failures and on unexpected spikes in remediation activity.
14. **Audit what changed.** CloudTrail shows the API calls made by the automation role.
15. **Consider the effect on running workloads.** Changing a security group or encryption setting might interrupt an application.
16. **Combine with preventive controls** (SCPs, IAM policies, CloudFormation Hooks). Remediation is a safety net **after** the fact, not the first line of defense.

---
