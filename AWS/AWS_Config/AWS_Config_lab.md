# AWS Config Hands-On Lab
### Detect and fix: S3 versioning, SSH open to the world, and IMDSv1

| | |
|---|---|
| **Level** | Beginner to intermediate |
| **Time** | About 2.5 hours |
| **Method** | AWS Management Console only. (A few short `curl` tests are typed inside the EC2 instance's browser terminal to prove IMDSv1/IMDSv2 behavior.) |
| **Cost** | Small (a few cents if you clean up the same day), but **not guaranteed free**. You pay for Config recording, rule evaluations, two or three tiny EC2 instances, a small SNS/EventBridge alert and S3 storage. Always finish Phase 14 (Cleanup). |
| **Console note** | AWS changes console layouts often. Button and menu labels below are accurate in intent, but may differ slightly on your screen. Rule names, runbook names and parameter names are exact. |

---

### Lab map

| Phase | What you do | Time |
|---|---|---|
| 1 | Pre-flight checks | 10 min |
| 2 | Turn on AWS Config | 15 min |
| 3 | Create the remediation IAM role | 15 min |
| 4 | Create the three non-compliant resources | 20 min |
| 5 | Baseline test: prove IMDSv1 works | 10 min |
| 6 | Create the three Config rules | 15 min |
| 7 | Observe detection, explore CIs and timeline | 20 min |
| 8 | **Prove Config detects ANY new EC2 instance launched with IMDSv1** | 25 min |
| 9 | Email alert when an IMDSv1 instance is detected (optional) | 20 min |
| 10 | Remediation: S3 automatic, IMDS manual, SSH automatic | 30 min |
| 11 | Drift test | 15 min |
| 12 | Break remediation on purpose (optional) | 15 min |
| 13 | Final verification checklist | 10 min |
| 14 | Cleanup (do not skip) | 15 min |

---

## 0. What you will build

You will deliberately create **three insecure/non-compliant resources**, let AWS Config **detect** them with managed rules, then **fix** them with SSM Automation remediation.

| # | Resource | The "bad" setting | Config managed rule | Remediation runbook (SSM Automation) | Remediation mode |
|---|---|---|---|---|---|
| 1 | **S3 bucket** | Versioning is off | `s3-bucket-versioning-enabled` | `AWS-ConfigureS3BucketVersioning` | **Automatic** (low risk) |
| 2 | **EC2 instances (any instance)** | IMDSv1 allowed (`HttpTokens = optional`) | `ec2-imdsv2-check` | `AWSConfigRemediation-EnforceEC2InstanceIMDSv2` | **Manual** (could break apps) |
| 3 | **Security group** | SSH (port 22) open to `0.0.0.0/0` | `restricted-ssh` | `AWS-DisablePublicAccessForSecurityGroup` | **Automatic** |

### Special focus: catching ANY EC2 instance launched with IMDSv1

You do not want to check instances one by one. The managed rule `ec2-imdsv2-check` evaluates **every EC2 instance in scope** and marks it **NON_COMPLIANT** when `HttpTokens` is `optional` (meaning IMDSv1 is allowed). Because the rule looks at the instance's *resulting configuration*, it does not matter who launched it or how (console, Auto Scaling, launch template, another tool).

You will prove this in **Phase 8** by launching new instances *after* the rule exists, without touching the rule, and watching Config flag the IMDSv1 one automatically. In **Phase 9** you will add an email alert.

> Config is **detective**. It flags the instance after it is recorded (usually within minutes). It does not block the launch. See Phase 8, step 8.8, for what it does and does not cover.

### Architecture

```
 You change a resource (S3 / EC2 / Security Group)
                    |
                    v
      Configuration recorder  --->  Configuration Item (CI)
                    |                        |
                    |                        +--> S3 delivery bucket (history files)
                    v
        Config Rule evaluates the CI
        COMPLIANT / NON_COMPLIANT
                    |
          (if NON_COMPLIANT)
                    v
   Remediation: manual click OR automatic
                    |
                    v
   SSM Automation runbook (uses IAM "AutomationAssumeRole")
                    |
                    v
   Resource is fixed -> Config records new CI -> rule re-evaluates -> COMPLIANT
```

### Concepts you will practice

Recorder, delivery channel, configuration items, relationships, resource timeline, change-triggered rules, managed rules, compliance results, manual vs automatic remediation, remediation parameters, `AutomationAssumeRole`, retries, remediation failure, drift detection, and Config vs CloudTrail.

---

## 1. Before you start

### 1.1 Requirements
- A **sandbox AWS account** (do not use a production account).
- An **IAM user or role with administrator-level access**. Do **not** use the root user.
- A web browser.

### 1.2 Safety rules for this lab
- The security group in this lab is **deliberately open to the internet on port 22**. That is the whole point of the exercise.
- To keep risk low: the EC2 instance has **no key pair**, **no data**, **no IAM role**, and lives for a few hours only.
- Do not reuse this security group or instance for anything else.
- Complete **Phase 14 (Cleanup)** when done.

### 1.3 Names used in this lab

Write your real values here. You will reuse them.

| Placeholder | Meaning | Your value |
|---|---|---|
| `<REGION>` | One AWS region used for **everything** in this lab | |
| `<ACCOUNT_ID>` | Your 12-digit AWS account ID | |
| `<BUCKET_1>` | `config-lab-<ACCOUNT_ID>-versioning` | |
| `<ROLE_ARN>` | ARN of the remediation role (created in Phase 3) | |
| `<SG_ID>` | ID of the lab security group (Phase 4) | |
| `<INSTANCE_ID>` | ID of the lab EC2 instance A (Phase 4) | |
| `<INSTANCE_B_ID>` | ID of instance B, IMDSv1 allowed (Phase 8) | |
| `<INSTANCE_C_ID>` | ID of instance C, IMDSv2 only (Phase 8) | |

> **Important:** AWS Config is **regional**. Use the **same region** from start to finish. Everything (Config, EC2, Systems Manager, S3 bucket) must be in `<REGION>`.

---

## Phase 1: Pre-flight checks (10 min)

**1.1 Sign in** to the AWS console with your admin IAM user/role.

**1.2 Choose the region.** In the top-right corner, open the region selector and pick `<REGION>`. Keep it for the entire lab.

**1.3 Note your account ID.** Click your account name in the top-right menu. Copy the **Account ID** (12 digits, may be shown with dashes; remove them) into the table above.

**1.4 Confirm a default VPC exists.** This matters, because the SSH remediation runbook can fail in non-default VPCs.
1. Open the **VPC** console -> **Your VPCs**.
2. Look for a VPC where **Default VPC** = **Yes**.
3. If none exists: choose **Actions** -> **Create default VPC** -> **Create default VPC**.

**1.5 Check whether AWS Config is already on in this region.**
1. Open the **AWS Config** console.
2. If you see a **Get started** page, Config is **not** enabled. Continue to Phase 2.
3. If you see a **Dashboard** with resource counts, Config is already enabled. Go to **Settings** and check which resource types are recorded. If an administrator or AWS Organization manages Config for you, **do not change those settings**. Ask your administrator, or use a different sandbox account.

**1.6 (Optional) Check the account-level IMDS default.**
1. Open the **EC2** console -> **Dashboard** -> **Account attributes** -> **Data protection and security**.
2. Note whether the account default for instance metadata is "no preference" or "IMDSv2 required".
3. This only changes the *default* shown when you launch an instance. In Phase 4 you will set the metadata option **explicitly** at launch, which overrides it.

**1.7 Check for other security groups that allow open SSH (blast radius).**
Automatic remediation in Phase 10C acts on **every** noncompliant security group in `<REGION>`, not only yours. The role from Phase 3 is allowed to change any security group.
1. **EC2** -> **Security Groups**.
2. For every group **other than** the default group and `config-lab-open-ssh-sg` (which does not exist yet), open the **Inbound rules** tab and look for SSH (port 22), or any rule allowing all traffic, from `0.0.0.0/0` or `::/0`.
3. If you find any, use an empty sandbox account or region instead. Or do Phase 10C in **manual** mode (the fallback is described there) so only your lab security group is touched.

**Expected result:** Default VPC exists, Config is not yet configured (or you know its current state), you know your account ID, and no other security group in the region allows open SSH.

---

## Phase 2: Turn on AWS Config (15 min)

We use **manual setup** so you can choose exactly what to record.

**2.1 Start setup.**
1. **AWS Config** console -> **Get started** (or **Settings** -> **Edit** if Config was partly set up).
2. Choose **Manual setup** (not "1-click setup").

**2.2 Recording strategy.**
1. **Recording strategy:** choose **Specific resource types**.
2. **Recording frequency:** choose **Continuous recording**.
3. Select exactly these resource types (use the search box):
   - **EC2 Instance** (`AWS::EC2::Instance`)
   - **EC2 SecurityGroup** (`AWS::EC2::SecurityGroup`)
   - **S3 Bucket** (`AWS::S3::Bucket`)
4. If you can find **Config ResourceCompliance** (`AWS::Config::ResourceCompliance`) in the list, also select it. It lets the resource timeline show compliance history later.

> **Why only these?** Rules need their resource types to be recorded. Recording just three types keeps cost and noise low. In real accounts you would usually record all supported types.

**2.3 IAM role (data governance).**
- Choose **Create AWS Config service-linked role**.
- This creates `AWSServiceRoleForConfig`, which lets Config read your resources.

**2.4 Delivery method (delivery channel).**
1. **Amazon S3 bucket:** choose **Create a bucket**.
2. Leave the generated bucket name (it looks like `config-bucket-<ACCOUNT_ID>`) and write it down. This is your **Config delivery bucket**.
3. **Amazon SNS topic:** leave **unchecked** (optional, not needed).

**2.5 Rules step.** If the wizard shows a page for choosing rules, **skip it**. You will add rules manually in Phase 6.

**2.6 Review and confirm.** Click **Next**, review the summary, then **Confirm**.

**2.7 Verify recording is on.**
1. Go to **AWS Config** -> **Settings**.
2. Check that **Recording** shows as **on / recording**, and that the recording strategy lists your three resource types.
3. Go to **Dashboard**. After a few minutes you should see resource counts for the types you selected.

**2.8 Look at the delivery bucket.**
1. Open **S3** -> your `config-bucket-<ACCOUNT_ID>` bucket.
2. Browse the folders. Config typically writes a small test object when it first verifies it can write to the bucket.
3. Configuration **history files** are delivered here every 6 hours for resource types that changed. You will not wait for that in this lab.

**Expected result:** Recorder on, delivery bucket exists, dashboard counts begin to appear.

---

## Phase 3: Create the remediation IAM role (15 min)

SSM Automation needs an IAM role (the **AutomationAssumeRole**) to perform the fixes. You will create a **least-privilege** policy and a role that SSM can assume.

### 3A. Create the policy

1. **IAM** console -> **Policies** -> **Create policy**.
2. Open the **JSON** tab/editor and **replace everything** with:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SsmAutomationBasics",
      "Effect": "Allow",
      "Action": [
        "ssm:StartAutomationExecution",
        "ssm:GetAutomationExecution"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3VersioningOnLabBucketsOnly",
      "Effect": "Allow",
      "Action": [
        "s3:PutBucketVersioning",
        "s3:GetBucketVersioning"
      ],
      "Resource": "arn:aws:s3:::config-lab-*"
    },
    {
      "Sid": "Imdsv2Enforcement",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:ModifyInstanceMetadataOptions"
      ],
      "Resource": "*"
    },
    {
      "Sid": "SecurityGroupSshCleanup",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeSecurityGroups",
        "ec2:RevokeSecurityGroupIngress"
      ],
      "Resource": "*"
    }
  ]
}
```

3. Click **Next**.
4. **Policy name:** `ConfigLabRemediationPolicy`.
5. Click **Create policy**.

> **Why `config-lab-*`?** It limits the S3 permission to buckets whose names start with `config-lab-`. This is the "least privilege / limited scope" idea from safe remediation design. That is why your lab bucket **must** be named `config-lab-...`.

### 3B. Create the role

1. **IAM** -> **Roles** -> **Create role**.
2. **Trusted entity type:** **AWS service**.
3. **Use case:** under "Use cases for other AWS services", choose **Systems Manager**, then select the **Systems Manager** option. Click **Next**.
4. On **Add permissions**, search for `ConfigLabRemediationPolicy`, tick it, click **Next**.
5. **Role name:** `ConfigLabRemediationRole`.
6. Click **Create role**.

### 3C. Verify the role

1. Open the role `ConfigLabRemediationRole`.
2. Copy the **ARN** (looks like `arn:aws:iam::<ACCOUNT_ID>:role/ConfigLabRemediationRole`) into your table as `<ROLE_ARN>`.
3. Open the **Trust relationships** tab. It must show the principal `ssm.amazonaws.com` with action `sts:AssumeRole`.
4. Open the **Permissions** tab. It must show `ConfigLabRemediationPolicy`.

**Expected result:** Role exists, trusts `ssm.amazonaws.com`, has the four permission blocks.

---

## Phase 4: Create the three non-compliant resources (20 min)

### 4.1 S3 bucket with versioning OFF

1. **S3** console -> **Create bucket**.
2. **Bucket type:** General purpose.
3. **Bucket name:** `config-lab-<ACCOUNT_ID>-versioning` (must start with `config-lab-`). Write it as `<BUCKET_1>`.
4. **AWS Region:** `<REGION>`.
5. Leave **Block all public access** **ON** (default and correct).
6. **Bucket Versioning:** keep **Disable** (default). This is the non-compliant setting.
7. Leave everything else default. Click **Create bucket**.

### 4.2 Security group with SSH open to the world

1. **EC2** console -> **Security Groups** -> **Create security group**.
2. **Security group name:** `config-lab-open-ssh-sg`
3. **Description:** `Config lab - SSH open on purpose`
4. **VPC:** select the **default VPC** (the one marked default).
5. **Inbound rules** -> **Add rule**:
   - **Type:** SSH
   - **Source:** Anywhere-IPv4 (`0.0.0.0/0`)
6. Leave **Outbound rules** as default (all traffic).
7. (Optional) Add a tag `Name` = `config-lab-open-ssh-sg`.
8. Click **Create security group**. The console may warn about `0.0.0.0/0`. That is expected.
9. Copy the **Security group ID** (`sg-...`) as `<SG_ID>`.

### 4.3 EC2 instance with IMDSv1 allowed

1. **EC2** console -> **Instances** -> **Launch instances**.
2. **Name:** `config-lab-instance`
3. **AMI:** **Amazon Linux 2023** (64-bit x86).
4. **Instance type:** `t3.micro` (or the smallest free-tier-eligible type shown).
5. **Key pair:** choose **Proceed without a key pair**. (You will connect with EC2 Instance Connect instead.)
6. **Network settings** -> **Edit**:
   - **VPC:** default VPC
   - **Subnet:** No preference
   - **Auto-assign public IP:** **Enable**
   - **Firewall (security groups):** **Select existing security group** -> choose `config-lab-open-ssh-sg`
7. **Storage:** leave default.
8. Expand **Advanced details** and scroll to the metadata settings:
   - **Metadata accessible:** **Enabled**
   - **Metadata version:** **V1 and V2 (token optional)**  <- this is the "IMDSv1 allowed" setting
   - **Metadata response hop limit:** leave default
   > Amazon Linux 2023 normally defaults to IMDSv2-only, so you **must** change this dropdown yourself.
9. Click **Launch instance**.
10. Open the instance, copy the **Instance ID** as `<INSTANCE_ID>`.

### 4.4 Wait for the instance to be ready

1. In **EC2** -> **Instances**, wait until **Instance state** = **Running** and **Status check** = **2/2 checks passed** (about 2 to 3 minutes).
2. Select the instance -> **Details** tab. Find **IMDSv2**. It must show **Optional**. (If it says **Required**, go back and fix it: **Actions** -> **Instance settings** -> **Modify instance metadata options** -> IMDSv2 **Optional**.)

**Expected result:** One bucket (versioning disabled), one security group (SSH open), one running instance (IMDSv2 = Optional) attached to that security group.

---

## Phase 5: Baseline test: prove IMDSv1 works (10 min)

You will connect to the instance and call the instance metadata service **without** a token (IMDSv1 style) and **with** a token (IMDSv2 style).

> This works only because the security group still allows SSH. That is why we remediate the security group **last**.

### 5.1 Connect with EC2 Instance Connect

1. **EC2** -> **Instances** -> select `config-lab-instance` -> **Connect**.
2. Choose the **EC2 Instance Connect** tab.
3. **Connection type:** **Connect using a Public IP**. **Username:** `ec2-user`.
4. Click **Connect**. A browser terminal opens.

If it fails: wait one more minute for the status checks, confirm the instance has a public IPv4 address, and confirm the security group allows SSH.

### 5.2 Test A: IMDSv1 (no token)

Type:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://169.254.169.254/latest/meta-data/instance-id
```

**Expected:** `200`.

See the actual value:

```bash
curl -s http://169.254.169.254/latest/meta-data/instance-id; echo
```

**Expected:** your instance ID (`i-...`). This proves **IMDSv1 works**.

### 5.3 Test B: IMDSv2 (with token)

```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
curl -s -o /dev/null -w "%{http_code}\n" -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id
```

**Expected:** `200`.

### 5.4 Record the baseline

| Test | Before remediation |
|---|---|
| A: no token | `200` |
| B: with token | `200` |

You can close the terminal tab. You will repeat these tests after remediation.

**Why this matters:** IMDSv1 uses plain requests, which makes it easier to abuse through vulnerabilities such as SSRF. IMDSv2 needs a session token first.

---

## Phase 6: Create the three Config rules (15 min)

Make sure you are still in `<REGION>`.

For each rule: **AWS Config** -> **Rules** -> **Add rule**.

### 6.1 Rule 1: `s3-bucket-versioning-enabled`

1. On **Specify rule type**, choose **AWS managed rule**.
2. Search `s3-bucket-versioning-enabled`, select it, click **Next**.
3. **Configure rule:**
   - **Name:** keep the default.
   - **Evaluation mode:** **Detective** on (turn proactive off if shown).
   - **Trigger type:** shows **Configuration changes**. Leave it.
   - **Scope of changes:** Resources. Resource type: S3 Bucket. Leave defaults.
   - **Parameters:** leave empty.
4. **Next** -> **Add rule**.

### 6.2 Rule 2: `ec2-imdsv2-check`

Repeat the same flow, searching for `ec2-imdsv2-check`. It has **no parameters**, and its trigger is **Configuration changes** on **EC2 Instance**.

**Scope (important for "detect ANY instance"):**
- **Resource type:** EC2 Instance.
- Leave **Tag key / Tag value** **empty**.
- Leave **Resource ID** **empty**.
- Empty filters mean **every** EC2 instance in this region is evaluated, including ones launched later.

### 6.3 Rule 3: `restricted-ssh`

Repeat the same flow, searching for `restricted-ssh`. It is triggered by **Configuration changes** on **EC2 SecurityGroup**.

> If you do not see your security group flagged later, confirm it is attached to the running instance. Some versions of this rule evaluate only security groups that are in use. Yours is attached, so it will be evaluated.

### 6.4 Confirm all three exist

**AWS Config** -> **Rules** should list: `s3-bucket-versioning-enabled`, `ec2-imdsv2-check`, `restricted-ssh`. Their **Compliance** column may say **Evaluating...** for a short time.

---

## Phase 7: Observe detection and explore Config (20 min)

### 7.1 See NON_COMPLIANT results

1. **AWS Config** -> **Rules**. Press refresh every minute. This usually takes a few minutes.
2. Open each rule and check **Resources in scope**.

**Expected results:**

| Rule | Your resource | Result |
|---|---|---|
| `s3-bucket-versioning-enabled` | `<BUCKET_1>` | **Noncompliant** |
| `ec2-imdsv2-check` | `<INSTANCE_ID>` | **Noncompliant** |
| `restricted-ssh` | `<SG_ID>` | **Noncompliant** |

> Other buckets or security groups in the account may also appear. The default security group has no SSH rule, so it should be compliant. Other buckets without versioning may also be noncompliant. That is correct behavior.

### 7.2 Inspect a Configuration Item (CI)

1. **AWS Config** -> **Resources**. Filter **Resource type** to **EC2 Instance**. Click your instance ID.
2. On the resource details page, review: **Configuration**, **Relationships**, **Changes**, and **Rules** (compliance).
3. Look for a link/button such as **View Configuration Item (JSON)** and open it.

Find these fields and write what you see:

| CI field | What to look for |
|---|---|
| `resourceId` | Your `i-...` ID |
| `resourceType` | `AWS::EC2::Instance` |
| `configurationItemStatus` | `OK` or `ResourceDiscovered` |
| `configurationItemCaptureTime` | When Config recorded it |
| `configuration` -> `metadataOptions` -> `httpTokens` | `optional` (this is exactly what the rule checks) |
| `relationships` | Links to your security group and other resources |

Repeat for the **S3 bucket** (look in `supplementaryConfiguration` for the versioning information) and the **security group** (look at `ipPermissions` for port 22 and `0.0.0.0/0`).

### 7.3 Explore Relationships

On the instance details page, open **Relationships**. You should see your security group. On the security group page, you should see the instance relationship. This is how Config supports **impact analysis**.

### 7.4 Resource Timeline

1. On the instance details page, click **Resource Timeline**.
2. Use the filters: **Configuration events**, **Compliance events**, **CloudTrail events**.
3. Note: right now you will mostly see the first recorded state and the compliance event. More entries appear after remediation.

### 7.5 (Optional) Advanced query

1. **AWS Config** -> **Advanced queries** (left menu).
2. Run:

```sql
SELECT
  resourceId,
  resourceType,
  configuration.metadataOptions.httpTokens
WHERE
  resourceType = 'AWS::EC2::Instance'
```

3. **Expected:** your instance with `httpTokens` = `optional`. If the nested field is blank, open the CI JSON from step 7.2 and adjust the field path to match exactly.

---

## Phase 8: Prove Config detects ANY new EC2 instance launched with IMDSv1 (25 min)

**Goal:** show that Config flags every instance that allows IMDSv1, including instances launched *after* the rule was created, with no action from you.

**Why it works (short version):**
1. The recorder is on for `AWS::EC2::Instance`, so every new or changed instance produces a Configuration Item.
2. The CI contains `configuration.metadataOptions.httpTokens`.
3. `ec2-imdsv2-check` is change-triggered, so each new CI is evaluated: `optional` -> **NON_COMPLIANT**, `required` -> **COMPLIANT**.

### 8.1 Confirm the rule covers all instances

1. **AWS Config** -> **Rules** -> `ec2-imdsv2-check`.
2. In the rule details, confirm the scope is **EC2 Instance** with **no tag filter and no resource ID filter**, and the trigger is **Configuration changes**.
3. If you added a filter earlier by mistake: **Actions** -> **Edit rule** -> clear the tag and resource ID fields -> **Save**.

### 8.2 Launch Instance B (IMDSv1 allowed)

1. **EC2** -> **Launch instances**.
2. **Name:** `config-lab-instance-b`
3. **AMI:** Amazon Linux 2023. **Instance type:** `t3.micro`.
4. **Key pair:** **Proceed without a key pair**.
5. **Network settings** -> **Edit**:
   - **VPC:** default VPC
   - **Auto-assign public IP:** **Disable**
   - **Firewall (security groups):** **Select existing security group** -> choose the group named **default**
   (This keeps instance B unreachable from the internet. You do not need to log in to it.)
6. **Advanced details**:
   - **Metadata accessible:** Enabled
   - **Metadata version:** **V1 and V2 (token optional)**
7. **Launch instance**. Copy the Instance ID as `<INSTANCE_B_ID>`.

### 8.3 Launch Instance C (IMDSv2 only, the "good" one)

Repeat 8.2 exactly, with these differences:
- **Name:** `config-lab-instance-c`
- **Metadata version:** **V2 only (token required)**

Copy the Instance ID as `<INSTANCE_C_ID>`.

### 8.4 Confirm the real settings in EC2

Wait until both instances are **Running**. For each: select it -> **Details** tab -> **IMDSv2**.

| Instance | Expected IMDSv2 value |
|---|---|
| `<INSTANCE_B_ID>` | **Optional** |
| `<INSTANCE_C_ID>` | **Required** |

### 8.5 Watch Config detect them (no action from you)

1. **AWS Config** -> **Rules** -> `ec2-imdsv2-check` -> **Resources in scope**. Refresh every minute. New instances usually appear within a few minutes.
2. **Expected results:**

| Instance | Result | Why |
|---|---|---|
| `<INSTANCE_ID>` (A) | **Noncompliant** | IMDSv1 allowed (not remediated yet) |
| `<INSTANCE_B_ID>` | **Noncompliant** | IMDSv1 allowed, detected automatically |
| `<INSTANCE_C_ID>` | **Compliant** | IMDSv2 required |

**What just happened:** you launched B and C and never edited the rule. Config discovered them, recorded their CIs, evaluated them, and flagged only the one that allows IMDSv1.

### 8.6 Inspect the evidence in the CIs

1. **AWS Config** -> **Resources** -> filter **EC2 Instance** -> open `<INSTANCE_B_ID>` -> open the **Configuration Item (JSON)** view.
2. Check these values:

| Field | Instance B | Instance C |
|---|---|---|
| `configurationItemStatus` | `ResourceDiscovered` (first CI) or `OK` | `ResourceDiscovered` or `OK` |
| `configuration` -> `metadataOptions` -> `httpTokens` | `optional` | `required` |

3. Open **Resource Timeline** for B. The first entry is when Config discovered it.

### 8.7 Build your "IMDSv1 inventory"

1. On the rule page, filter **Resources in scope** by **Noncompliant**. This list is every instance in the region that allows IMDSv1.
2. (Optional) **AWS Config** -> **Advanced queries**, run:

```sql
SELECT
  resourceId,
  configuration.metadataOptions.httpTokens
WHERE
  resourceType = 'AWS::EC2::Instance'
  AND configuration.metadataOptions.httpTokens = 'optional'
```

3. **Expected:** instances A and B (and any other IMDSv1 instance in the region). If the field path returns nothing, copy the exact path from the CI JSON in step 8.6.

### 8.8 What this does and does not cover

| Covered | Not covered |
|---|---|
| Every EC2 instance in this region and account while the recorder is on for EC2 Instance | **Other regions.** Enable Config and the rule in each region |
| Instances launched by any method (console, Auto Scaling, launch templates, other tools) | **Other accounts.** Use an aggregator or organization rules for many accounts |
| Instances that already existed when recording started (they are discovered and evaluated) | **Blocking the launch.** Config is detective, not preventive |
| Instances changed later from IMDSv2 to IMDSv1 (new CI, re-evaluated) | **Instant detection.** There is a delay of a few minutes |

**To prevent IMDSv1 instead of only detecting it**, use separate preventive controls, for example the EC2 **account-level IMDS default set to IMDSv2 required** (EC2 -> Dashboard -> Account attributes -> Data protection and security), or an SCP/IAM policy using the condition key `ec2:MetadataHttpTokens`. Keep Config as the safety net that proves nothing slipped through.

---

## Phase 9 (Optional): Get an email when an IMDSv1 instance is detected (20 min)

Config sends compliance-change events to **Amazon EventBridge**. You will route the NON_COMPLIANT events for `ec2-imdsv2-check` to an **SNS email topic**.

> The SNS topic **must be in the same region as AWS Config**.

### 9A. Create the SNS topic and email subscription

1. **SNS** console -> **Topics** -> **Create topic**.
2. **Type:** Standard. **Name:** `config-lab-imds-alerts`. Click **Create topic**.
3. Open the topic -> **Create subscription**:
   - **Protocol:** Email
   - **Endpoint:** your email address
4. Click **Create subscription**.
5. Open your inbox, find the **AWS Notification - Subscription Confirmation** email, and click **Confirm subscription**. (Check spam if it does not arrive.)
6. Back in SNS, the subscription status must be **Confirmed**.

### 9B. Create the EventBridge rule

1. **EventBridge** console -> **Rules** -> **Create rule**.
2. **Name:** `config-lab-imdsv1-detected`. **Event bus:** default. **Rule type:** **Rule with an event pattern**. Click **Next**.
3. **Event source:** **AWS events or EventBridge partner events**.
4. **Creation method:** **Custom pattern (JSON editor)**. Paste:

```json
{
  "source": ["aws.config"],
  "detail-type": ["Config Rules Compliance Change"],
  "detail": {
    "configRuleName": ["ec2-imdsv2-check"],
    "newEvaluationResult": {
      "complianceType": ["NON_COMPLIANT"]
    }
  }
}
```

5. Click **Next**.
6. **Target types:** AWS service. **Select a target:** **SNS topic**. **Topic:** `config-lab-imds-alerts`.
7. Click **Next** -> **Next** -> **Create rule**.

### 9C. Trigger a real alert

Events fire when a resource's compliance **changes** to NON_COMPLIANT. Instances A and B were already noncompliant before this rule existed, so they do not trigger it. Use instance C:

1. **EC2** -> select `config-lab-instance-c` -> **Actions** -> **Instance settings** -> **Modify instance metadata options**.
2. Set **IMDSv2** to **Optional** -> **Save**.
3. Wait a few minutes. Config records the change and `ec2-imdsv2-check` flips C from Compliant to **Noncompliant**.
4. **Expected:** an email from the SNS topic. The body is raw JSON. Find `configRuleName` (`ec2-imdsv2-check`), `resourceId` (`<INSTANCE_C_ID>`) and `complianceType` (`NON_COMPLIANT`).
5. (Optional) Set C back to **Required** to return it to Compliant.

If no email arrives:
- Confirm the subscription status is **Confirmed**.
- Confirm the topic and Config are in the **same region**.
- Check the EventBridge rule is **Enabled** and the pattern JSON was pasted exactly.
- Check the SNS topic's **Access policy** allows `events.amazonaws.com` to publish. (The console normally adds this when you pick the SNS target.)

---

## Phase 10: Remediation (30 min)

For every remediation you configure, open the rule first:
**AWS Config** -> **Rules** -> click the rule -> **Actions** -> **Manage remediation**.

### 10A. S3: automatic remediation

1. Open rule `s3-bucket-versioning-enabled` -> **Actions** -> **Manage remediation**.
2. **Remediation method:** **Automatic remediation**.
3. **Retries in case of failure:** keep the defaults (up to **5** attempts within **60** seconds).
4. **Remediation action:** search and select **`AWS-ConfigureS3BucketVersioning`**.
5. **Resource ID parameter:** select **`BucketName`**. (Config will fill in the non-compliant bucket's name automatically.)
6. **Parameters** (add these):
   - `AutomationAssumeRole` = `<ROLE_ARN>`
   - `VersioningState` = `Enabled`
7. **Save changes**.

> **Remember from the guide:** turning on automatic remediation also triggers remediation for resources that are **already** noncompliant. So the fix should start within about a minute. If other buckets in your account (not named `config-lab-...`) are also noncompliant, the runbook will be attempted for them too, but **will fail with access denied** because your role is limited to `config-lab-*`. That is expected and harmless in a sandbox, and it shows least privilege working.

**Verify (give it a few minutes):**

1. **Systems Manager** console -> **Automation** -> **Executions**. Find a recent execution of `AWS-ConfigureS3BucketVersioning` with status **Success**. Open it and review the step results.
2. **S3** -> `<BUCKET_1>` -> **Properties** -> **Bucket Versioning**. **Expected:** **Enabled**.
3. **AWS Config** -> **Rules** -> `s3-bucket-versioning-enabled`. After Config records the bucket change, `<BUCKET_1>` becomes **Compliant**. This can take a few minutes. Refresh.

### 10B. EC2: manual remediation (IMDSv2)

**Configure the action (manual):**

1. Open rule `ec2-imdsv2-check` -> **Actions** -> **Manage remediation**.
2. **Remediation method:** **Manual remediation**.
3. **Remediation action:** **`AWSConfigRemediation-EnforceEC2InstanceIMDSv2`**.
4. **Resource ID parameter:** **`InstanceId`**.
5. **Parameters:**
   - `AutomationAssumeRole` = `<ROLE_ARN>`
   - Leave `HttpPutResponseHopLimit` at its default of `0` (means "do not change").
6. **Save changes**.

**Nothing happens yet.** That is the point of manual mode. Confirm: the rule still shows `<INSTANCE_ID>` as **Noncompliant**, and the instance's IMDSv2 setting is still **Optional**.

**Run the fix yourself:**

1. On the rule page, in **Resources in scope**, **select** `<INSTANCE_ID>` (instance A). You may also select `<INSTANCE_B_ID>` to fix it in the same click, or run **Remediate** for it separately afterwards.
2. Click **Remediate**.
3. **Systems Manager** -> **Automation** -> **Executions**: wait for `AWSConfigRemediation-EnforceEC2InstanceIMDSv2` to show **Success**.
4. **EC2** -> your instance -> **Details** -> **IMDSv2** should now show **Required**.
5. Back in **AWS Config**, after a few minutes the rule shows `<INSTANCE_ID>` as **Compliant**.

**Re-run the baseline tests (SSH is still open, so you can connect):**

1. Connect again with EC2 Instance Connect (as in step 5.1).
2. Test A (no token):

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://169.254.169.254/latest/meta-data/instance-id
```

**Expected now:** `401` (IMDSv1 is rejected).

3. Test B (with token):

```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
curl -s -o /dev/null -w "%{http_code}\n" -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id
```

**Expected:** `200`.

| Test | Before | After |
|---|---|---|
| A: no token (IMDSv1) | `200` | **`401`** |
| B: with token (IMDSv2) | `200` | `200` |

### 10C. Security group: automatic remediation (do this last)

> After this step the SSH rule is removed, so you will no longer be able to connect with EC2 Instance Connect. That is why this is last.

> **Blast radius:** automatic remediation runs for **every** noncompliant security group in the region, not just yours. You checked for others in step 1.7. If you found any and could not avoid them, use this **manual fallback** instead: in step 2 below choose **Manual remediation**, save, then on the rule page select **only** `<SG_ID>` and click **Remediate**. All other steps stay the same.

1. Open rule `restricted-ssh` -> **Actions** -> **Manage remediation**.
2. **Remediation method:** **Automatic remediation**.
3. **Retries in case of failure:** defaults (5 attempts, 60 seconds).
4. **Remediation action:** **`AWS-DisablePublicAccessForSecurityGroup`**.
5. **Resource ID parameter:** **`GroupId`**.
6. **Parameters:**
   - `AutomationAssumeRole` = `<ROLE_ARN>`
   - Leave `IpAddressToBlock` empty.
7. **Save changes**.

**Verify:**

1. **Systems Manager** -> **Automation** -> **Executions**: `AWS-DisablePublicAccessForSecurityGroup` should show **Success**.
2. **EC2** -> **Security Groups** -> `config-lab-open-ssh-sg` -> **Inbound rules**. **Expected:** the SSH `0.0.0.0/0` rule is **gone**.
3. **AWS Config** -> Rule `restricted-ssh`: `<SG_ID>` becomes **Compliant** after Config records the change.
4. (Optional) Try **EC2 Instance Connect** again. **Expected:** it now fails to connect, because port 22 is no longer open. This shows the real-world effect of automatic remediation.

### 10D. See who made the changes (Config vs CloudTrail)

1. **AWS Config** -> **Resources** -> your security group -> **Resource Timeline** -> filter **CloudTrail events**. You should see a `RevokeSecurityGroupIngress` event. (CloudTrail data can take several minutes to appear, and the link works for recent events.)
2. Open **CloudTrail** -> **Event history**. Filter **Event name** = `RevokeSecurityGroupIngress`. Open the event and look at **userIdentity**. It shows an **assumed role session** for `ConfigLabRemediationRole`, not a person.
3. Do the same for `PutBucketVersioning` and `ModifyInstanceMetadataOptions`.

**This is the key idea:** Config shows **what** the resource became. CloudTrail shows **who/what** made the API call.

---

## Phase 11: Drift test (15 min)

This proves Config keeps watching. You will break things again and watch Config detect and fix them.

### 11A. S3 drift (automatic fix)

1. **S3** -> `<BUCKET_1>` -> **Properties** -> **Bucket Versioning** -> **Edit** -> **Suspend** -> **Save**.
2. Watch **AWS Config** -> Rule `s3-bucket-versioning-enabled`. It goes **Noncompliant**, then automatic remediation re-enables versioning, then back to **Compliant**. This takes a few minutes.
3. Open the bucket's **Resource Timeline**. You should now see several CIs (original, enabled, suspended, enabled again).

### 11B. Security group drift (automatic fix)

1. **EC2** -> **Security Groups** -> `config-lab-open-ssh-sg` -> **Edit inbound rules** -> **Add rule**: SSH, `0.0.0.0/0` -> **Save rules**.
2. Watch Config detect it (**Noncompliant**) and automatic remediation remove the rule again.

### 11C. IMDS drift (manual fix, so it stays noncompliant)

1. **EC2** -> instance -> **Actions** -> **Instance settings** -> **Modify instance metadata options** -> set IMDSv2 to **Optional** -> **Save**.
2. Watch `ec2-imdsv2-check` go **Noncompliant**. Because remediation is **manual**, it **stays** noncompliant until you click **Remediate** again.
3. Compare this with 11A and 11B. This is the difference between automatic and manual remediation in practice.
4. (Optional) Run **Remediate** again from the rule page to fix it.

---

## Phase 12 (Optional): Break remediation on purpose (15 min)

Seeing a failure once makes you much better at debugging real ones.

1. **IAM** -> **Policies** -> `ConfigLabRemediationPolicy` -> **Edit** -> in the JSON, **remove** the line `"s3:PutBucketVersioning",` (keep the JSON valid) -> save.
2. **S3** -> **Create bucket** named `config-lab-<ACCOUNT_ID>-fail-test`, versioning **disabled**.
3. Wait for `s3-bucket-versioning-enabled` to mark it **Noncompliant**. Automatic remediation will start and **fail**.
4. Investigate like you would in real life:
   1. **AWS Config** -> Rule -> the bucket -> look at the **remediation status** (failed / error).
   2. **Systems Manager** -> **Automation** -> **Executions** -> open the failed execution -> find the **failed step** and read the error message. You should see an **access denied** type error.
5. Note the retry behavior. Config retries failures. If it reaches the maximum number of failed attempts inside the retry window, Config **adds a remediation exception** for that resource and stops automatic remediation for it.
6. **Fix it:**
   1. Restore the `"s3:PutBucketVersioning",` line in `ConfigLabRemediationPolicy`.
   2. Try **Remediate** on the bucket from the rule page.
   3. If remediation is blocked because an exception was added, simply **delete the fail-test bucket** and create a new one with the same settings to continue.

**What you learned:** failures live in the **SSM Automation execution**, not in Config alone. The fix is almost always **permissions or parameters**. Exceptions exist to stop endless retry loops.

---

