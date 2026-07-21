# AWS Hands-on Lab: Start and Stop a Specific EC2 Instance Using Lambda

**Difficulty:** Beginner–Intermediate
**Estimated Time:** 40–60 Minutes

**AWS Services Used:**
- AWS Lambda
- Amazon EventBridge (Scheduled Rules)
- IAM
- EC2
- CloudWatch Logs

---

## Scenario

You have **one specific EC2 instance** (e.g., a dev server) that you want to start automatically at 9 AM and stop automatically at 8 PM on weekdays — no tags, no scanning your whole account, just that one instance ID, targeted directly.

This is the simplest version of EC2 scheduling: the instance ID is passed straight into the Lambda function or read from an environment variable. It's a good starting point before scaling up to tag-based scheduling across many instances (a natural next step once this pattern is comfortable).

---

## Architecture

```
        EventBridge Rule (Start)              EventBridge Rule (Stop)
        cron: 03:30 UTC Mon-Fri               cron: 14:30 UTC Mon-Fri
        (09:00 AM IST)                        (08:00 PM IST)
                │                                       │
                └───────────────┬───────────────────────┘
                                 ▼
                     ┌────────────────────────┐
                     │   Lambda Function       │
                     │   ec2-start-stop        │
                     │   (Python 3.12)         │
                     └────────────┬─────────────┘
                                  │
                    IAM Execution Role
                    ec2:StartInstances / StopInstances
                    scoped to ONE instance ARN
                    logs:PutLogEvents
                                  │
                                  ▼
                        Specific EC2 Instance
                        i-0123456789abcdef0
                                  │
                                  ▼
                     CloudWatch Logs
                     /aws/lambda/ec2-start-stop
```

---

## Concept Deep Dive

**Instance ID vs. tag-based targeting.** Instead of searching your account for matching tags, this Lambda is told exactly which instance to act on — either hardcoded in the environment variables, or passed in the EventBridge target's input JSON. This is simpler to reason about and easier to lock down with IAM, at the cost of needing a code/config change if you ever want to add a second instance.

**IAM scoped to one instance ARN.** Because you know the exact instance ID upfront, the IAM policy can reference that instance's ARN directly (`arn:aws:ec2:ap-south-1:<account-id>:instance/i-0123...`) instead of using a tag `Condition`. This is the tightest possible permission scope — the Lambda literally cannot act on any other instance, even if the code had a bug.

**Same start/stop function, different input.** As with the tag-based version, one function handles both directions based on an `action` field in the event — no need for two separate Lambdas.

---

## Prerequisites

- AWS Account
- IAM User with permissions for Lambda, EventBridge, and IAM
- An existing EC2 instance — **copy its Instance ID** (e.g., `i-0123456789abcdef0`) from the EC2 console; you'll need it in Step 2 and Step 4
- Region selected (Mumbai / `ap-south-1` used in this guide)

---

## Step 1 — Note the Target Instance ID

Navigate to **EC2 → Instances**, select the instance you want to schedule, and copy its **Instance ID** from the details panel.

No tagging is required for this version — the ID itself is the target.

---

## Step 2 — Create the IAM Execution Role

Navigate to **IAM → Roles → Create Role**

| Setting | Value |
| ------- | ----- |
| Trusted entity | AWS Service |
| Use case | Lambda |

Name: `ec2-start-stop-lambda-role`

Skip attaching a managed policy — you'll add a scoped inline policy in Step 3.

---

## Step 3 — Attach an Inline Policy Scoped to That One Instance

Open `ec2-start-stop-lambda-role` → **Add permissions → Create inline policy** → **JSON**

Replace `<ACCOUNT_ID>` and `<INSTANCE_ID>` with your own values:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "StartStopSpecificInstance",
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:DescribeInstances"
      ],
      "Resource": "arn:aws:ec2:ap-south-1:<ACCOUNT_ID>:instance/<INSTANCE_ID>"
    },
    {
      "Sid": "Logging",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

> `ec2:DescribeInstances` technically doesn't support resource-level restriction and AWS will apply it account-wide regardless of the ARN listed — that's expected and fine, since it's a read-only, low-risk action. `StartInstances`/`StopInstances` are the ones that matter, and those are correctly scoped to just this one instance.

Name the policy `ec2-start-stop-policy` and save.

---

## Step 4 — Create the Lambda Function

Navigate to **Lambda → Create Function**

| Setting | Value |
| ------- | ----- |
| Name | `ec2-start-stop` |
| Runtime | Python 3.12 |
| Architecture | x86_64 |
| Execution role | Use existing role → `ec2-start-stop-lambda-role` |

Create function, then replace `lambda_function.py` with:

```python
import boto3
import os

ec2 = boto3.client('ec2')

INSTANCE_ID = os.environ['INSTANCE_ID']  # e.g. i-0123456789abcdef0


def lambda_handler(event, context):
    action = event.get('action')
    if action not in ('start', 'stop'):
        raise ValueError("Event must include \"action\": \"start\" or \"stop\"")

    # Check current state so we don't call start on an already-running
    # instance or stop on an already-stopped one.
    response = ec2.describe_instances(InstanceIds=[INSTANCE_ID])
    state = response['Reservations'][0]['Instances'][0]['State']['Name']
    print(f"{INSTANCE_ID} current state: {state}")

    if action == 'stop' and state != 'running':
        print(f"Skipping stop — instance is already '{state}'.")
        return {'action': action, 'instance': INSTANCE_ID, 'skipped': True, 'state': state}

    if action == 'start' and state != 'stopped':
        print(f"Skipping start — instance is already '{state}'.")
        return {'action': action, 'instance': INSTANCE_ID, 'skipped': True, 'state': state}

    if action == 'stop':
        ec2.stop_instances(InstanceIds=[INSTANCE_ID])
    else:
        ec2.start_instances(InstanceIds=[INSTANCE_ID])

    print(f"{action.capitalize()} triggered for {INSTANCE_ID}")
    return {'action': action, 'instance': INSTANCE_ID, 'skipped': False}
```

Click **Deploy**.

---

## Step 5 — Set the Instance ID as an Environment Variable

Go to **Configuration → Environment Variables → Edit → Add environment variable**:

| Key | Value |
| --- | ----- |
| INSTANCE_ID | `i-0123456789abcdef0` *(your actual instance ID from Step 1)* |

Save.

> Keeping the instance ID as an environment variable (rather than hardcoded in the code) means you can point this same function at a different instance later just by editing the config — no redeploy needed.

---

## Step 6 — Test Manually

Go to the **Test** tab → create two test events:

**Stop test:**
```json
{ "action": "stop" }
```

**Start test:**
```json
{ "action": "start" }
```

Run the **stop** test and confirm in the EC2 console that the instance transitions to `stopping` → `stopped`. Run the **start** test and confirm it goes back to `running`. Check **CloudWatch Logs** for the printed state messages.

---

## Step 7 — Create the EventBridge Rule for Start

Navigate to **EventBridge → Rules → Create Rule**

| Setting | Value |
| ------- | ----- |
| Name | `ec2-start-stop-start-rule` |
| Rule type | Schedule |
| Cron expression | `cron(30 3 ? * MON-FRI *)` |

> 03:30 UTC = 09:00 AM IST. EventBridge cron is always UTC.

Target: **AWS Lambda function** → `ec2-start-stop`

**Configure target input → Constant (JSON text):**
```json
{ "action": "start" }
```

Create rule. (The console auto-grants the Lambda invoke permission for this rule — no separate step needed here.)

---

## Step 8 — Create the EventBridge Rule for Stop

Repeat Step 7 with:

| Setting | Value |
| ------- | ----- |
| Name | `ec2-start-stop-stop-rule` |
| Cron expression | `cron(30 14 ? * MON-FRI *)` |

> 14:30 UTC = 08:00 PM IST.

Target input:
```json
{ "action": "stop" }
```

---

## Step 9 — Verify

Re-run the Step 6 test events, or wait for the next scheduled trigger. Confirm the instance state changes at the expected time and check `/aws/lambda/ec2-start-stop` in CloudWatch Logs for the state-check output.


---

## Edge Cases & Failure Scenarios

| Scenario | What Happens | Handling |
| -------- | ------------- | -------- |
| Stop rule fires while instance is already stopped | Function checks current state first and skips the API call | Logged as `skipped: true` — no error, no wasted API call |
| Start rule fires while instance is already running | Same as above — skipped cleanly | No duplicate start attempts |
| Instance is terminated and its ID no longer exists | `describe_instances` raises `InvalidInstanceID.NotFound` | Function will error — update or remove `INSTANCE_ID` if the instance is permanently gone |
| Instance ID changes (e.g., replaced instance) | Old ID in the env var no longer matches anything | Update the `INSTANCE_ID` environment variable — no redeploy required |
| You want to schedule a second instance | This pattern only handles one ID | Either duplicate this whole stack with a new function name/role, or graduate to the tag-based scheduler pattern instead |
| Someone manually starts the instance outside the schedule | No conflict — function only acts based on the schedule + current state check | The next scheduled stop will still work correctly since it checks live state, not a cached assumption |

---

## Troubleshooting

| Issue | Possible Cause | Solution |
| ----- | --------------- | -------- |
| `AccessDenied` on `StartInstances`/`StopInstances` | ARN in the IAM policy doesn't match the instance's actual ARN/region/account | Double-check region, account ID, and instance ID in the policy `Resource` field |
| `InvalidInstanceID.NotFound` | Wrong instance ID in the environment variable, or instance was terminated | Verify `INSTANCE_ID` matches an instance that currently exists |
| EventBridge rule enabled but nothing happens | Missing invoke permission (CLI/Terraform path) | Add `aws_lambda_permission` / `aws lambda add-permission` for that rule's ARN |
| Function runs on schedule but state doesn't change | Rule fired while instance was already in target state | Check CloudWatch Logs for `"skipped": true` — this is expected behavior, not a bug |
| Wrong trigger time | UTC vs IST conversion mistake | IST is UTC+5:30 — subtract 5:30 from the desired IST time for the cron value |

---

## Cheat Sheet

| Task | Value / Command |
| ---- | ---------------- |
| Target one instance | Set `INSTANCE_ID` env var — no tags needed |
| IAM scope | `Resource: arn:aws:ec2:<region>:<account>:instance/<id>` |
| 9:00 AM IST in UTC cron | `cron(30 3 ? * MON-FRI *)` |
| 8:00 PM IST in UTC cron | `cron(30 14 ? * MON-FRI *)` |
| Manual test payload | `{"action":"start"}` / `{"action":"stop"}` |
| Log group | `/aws/lambda/ec2-start-stop` |
| Change target instance later | Edit the `INSTANCE_ID` env var, no redeploy |

---

## Mastery Checklist

- [ ] Can explain when to use a specific-instance-ID approach vs. a tag-based approach
- [ ] Can scope an IAM policy to a single instance ARN
- [ ] Understands why the function checks current state before calling start/stop
- [ ] Can convert IST to UTC for EventBridge cron expressions correctly
- [ ] Knows that console auto-grants Lambda invoke permission but CLI/Terraform require it explicitly
- [ ] Can explain the tradeoff of this pattern (simple, tightly scoped) vs. tag-based (scales, less IAM precision per-instance)

---

## Cleanup

Delete in this order:

1. Delete both EventBridge rules (`ec2-start-stop-start-rule`, `ec2-start-stop-stop-rule`).
2. Delete the Lambda function `ec2-start-stop`.
3. Delete the IAM role `ec2-start-stop-lambda-role`.
4. Stop or terminate the test instance if it was created solely for this lab.

---

## Learning Outcomes

After completing this lab, you will understand how to:

- Trigger Lambda on a schedule to start/stop a specific, known EC2 instance.
- Scope an IAM policy to a single resource ARN for tighter security than a tag-based `Condition`.
- Check live instance state before acting, to avoid redundant or conflicting API calls.
- Recognize when this simpler pattern is the right fit, and when to graduate to tag-based scheduling for multiple instances.
