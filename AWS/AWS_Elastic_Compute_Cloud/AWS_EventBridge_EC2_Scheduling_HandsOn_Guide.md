

# Schedule EC2 Start & Stop using Amazon EventBridge Scheduler (End-to-End)

> This guide follows the current AWS Console workflow.

## Objective

Automatically: - Start an EC2 instance every day at **09:00 AM IST** -
Stop the same EC2 instance every day at **07:00 PM IST**

------------------------------------------------------------------------

# Architecture

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/5e773002-263e-4b71-9185-d0767ac88129" />

------------------------------------------------------------------------

# Prerequisites

-   AWS Account
-   EC2 Instance
-   IAM permissions
-   Region selected correctly (same as EC2)

Example Instance ID :

``` text
i-04e8cc881de926954
```

------------------------------------------------------------------------

# PART 1 -- Create IAM Execution Role

## Step 1

AWS Console

``` text
IAM
→ Roles
→ Create role
```

## Step 2

Trusted Entity

Select

``` text
Custom trust policy
```

Paste

``` json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Principal":{
        "Service":"scheduler.amazonaws.com"
      },
      "Action":"sts:AssumeRole"
    }
  ]
}
```

Click **Next**

------------------------------------------------------------------------

## Step 3

Search policy

``` text
AmazonEC2FullAccess
```

Select it.

> Lab purpose only.
> For production, replace this with a least-privilege policy allowing
> only `ec2:StartInstances` and `ec2:StopInstances` on the required instance.

<img width="1643" height="832" alt="image" src="https://github.com/user-attachments/assets/12847ffb-40e0-4e7c-bbaa-a79f7b252dc1" />

Click **Next**

------------------------------------------------------------------------

## Step 4

Role Name

``` text
instance_Sch
```

Click

``` text
Create Role
```

------------------------------------------------------------------------

# PART 2 -- Create START Schedule

## Step 1

Open

``` text
Amazon EventBridge
→ Scheduler
→ Create Schedule
```

------------------------------------------------------------------------

## Schedule Details

  Field            Value
  ---------------- ------------------------------------
  Schedule Name    Start-EC2-Daily
  Description      Automatically starts EC2 every day
  Schedule Group   default
  Occurrence       Recurring
  Time Zone        Asia/Calcutta
  Schedule Type    Cron-based

Cron Fields

  Min   Hr   Day   Month   Week   Year
  ----- ---- ----- ------- ------ ------
  0     9    \*    \*      ?      \*

Meaning:

``` text
Every day at 09:00 AM IST
```

Flexible Time Window

``` text
OFF
```

Timeframe

``` text
No End Date
```

Click **Next**

------------------------------------------------------------------------

## Select Target

Choose

``` text
All APIs
```

Search

``` text
StartInstances
```

Select

``` text
Amazon EC2 → StartInstances
```

Payload

``` json
{
  "InstanceIds":[
    "Instance ID"
  ]
}
```

Click **Next**

------------------------------------------------------------------------

## Settings

Schedule State

``` text
Enabled
```

Action after completion

``` text
None
```

Retry

``` text
Disabled
```

Dead Letter Queue

``` text
None
```

Encryption

``` text
AWS Managed Key
```

Permissions

``` text
Use Existing Role
```

Select Your Role

``` text
instance_Sch
```

Click **Next**

------------------------------------------------------------------------

## Review

Verify:

-   Schedule Name
-   Cron
-   API = StartInstances
-   Instance ID
-   Role = instance_Sch

Click

``` text
Create Schedule
```

------------------------------------------------------------------------

# PART 3 -- Create STOP Schedule

Repeat the same process.

Only change:

  Field           Value
  --------------- -----------------
  Schedule Name   Stop-EC2-Daily
  API             StopInstances
  Cron            0 19 \* \* ? \*

Meaning

``` text
Every day at 07:00 PM IST
```

Payload

``` json
{
  "InstanceIds":[
    "Your Instance ID"
  ]
}
```

Use role

``` text
instance_Sch
```

Click

``` text
Create Schedule
```

------------------------------------------------------------------------

# Verification

Go to

``` text
Amazon EventBridge
→ Scheduler
```

You should see

``` text
Start-EC2-Daily   Enabled
Stop-EC2-Daily    Enabled
```
<img width="3339" height="1036" alt="image" src="https://github.com/user-attachments/assets/0d225c08-c484-4479-9b40-06337140bdea" />


------------------------------------------------------------------------

# Testing

## Start

1.  Stop EC2 manually.
2.  Change cron to 2 minutes ahead.
3.  Wait.
4.  Verify

``` text
Stopped
 ↓
Pending
 ↓
Running
```

## Stop

1.  Start EC2 manually.
2.  Change cron to 2 minutes ahead.
3.  Wait.
4.  Verify

``` text
Running
 ↓
Stopping
 ↓
Stopped
```

Restore production cron afterwards.

------------------------------------------------------------------------

# Troubleshooting

  Problem               Fix
  --------------------- ----------------------------------------------
  AccessDenied          Check IAM policy
  Cannot assume role    Verify trust policy
  Wrong instance        Check payload JSON
  Schedule never runs   Verify region, cron, timezone, enabled state
  Role not visible      Refresh role list after creation

------------------------------------------------------------------------

# Production Best Practices

-   Use least-privilege IAM policy.
-   Restrict to a specific EC2 ARN.
-   Keep Flexible Time Window OFF.
-   Monitor executions with CloudWatch.
-   Review IAM permissions periodically.

------------------------------------------------------------------------

# Final Outcome

  Schedule          API              Time
  ----------------- ---------------- --------------
  Start-EC2-Daily   StartInstances   09:00 AM IST
  Stop-EC2-Daily    StopInstances    07:00 PM IST

Lab completed successfully.
