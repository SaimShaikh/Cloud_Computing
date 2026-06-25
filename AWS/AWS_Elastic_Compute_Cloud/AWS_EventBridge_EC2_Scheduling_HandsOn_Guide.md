# AWS Hands-on Lab: Schedule EC2 Start & Stop Using Amazon EventBridge Scheduler

## Objective

Automatically start an EC2 instance every day at **9:00 AM IST** and
stop it every day at **7:00 PM IST** using Amazon EventBridge Scheduler.

------------------------------------------------------------------------

# Prerequisites

-   AWS Account
-   One EC2 instance
-   IAM permissions to create roles and schedules
-   Region: `ap-south-1` (or the region where your EC2 instance exists)

Example Instance ID:

``` text
i-04e8cc881de926954
```

------------------------------------------------------------------------

# Architecture

``` text
Amazon EventBridge Scheduler
            │
            ▼
   IAM Execution Role
            │
            ▼
 Amazon EC2 API (StartInstances / StopInstances)
            │
            ▼
      EC2 Instance
```

------------------------------------------------------------------------

# Part 1 -- Create IAM Execution Role

## Step 1

Go to:

``` text
IAM
→ Roles
→ Create Role
```

Choose:

``` text
Trusted Entity Type
→ Custom Trust Policy
```

Paste:

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "scheduler.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Click **Next**.

## Step 2

Attach policy:

``` text
AmazonEC2FullAccess
```

> For production, replace this with a least-privilege policy allowing
> only `ec2:StartInstances` and `ec2:StopInstances` on the required
> instance.

## Step 3

Role Name:

``` text
instance_Sch
```

Click **Create Role**.

------------------------------------------------------------------------

# Part 2 -- Create Start Schedule

## Schedule Details

  Setting                Value
  ---------------------- -----------------
  Schedule Name          Start-EC2-Daily
  Schedule Group         default
  Occurrence             Recurring
  Schedule Type          Cron
  Cron                   `0 9 * * ? *`
  Time Zone              Asia/Calcutta
  Flexible Time Window   OFF

## Target

-   All APIs
-   Amazon EC2
-   API: `StartInstances`

Payload:

``` json
{
  "InstanceIds": [
    "i-04e8cc881de926954"
  ]
}
```

## Settings

  Setting                   Value
  ------------------------- -----------------
  Enable Schedule           Enabled
  Action After Completion   None
  Retry                     Disabled
  Dead Letter Queue         None
  Encryption                AWS Managed Key
  Execution Role            instance_Sch

Review and click **Create schedule**.

------------------------------------------------------------------------

# Part 3 -- Create Stop Schedule

Use the same settings except:

  Setting         Value
  --------------- ----------------
  Schedule Name   Stop-EC2-Daily
  API             StopInstances
  Cron            `0 19 * * ? *`

Payload:

``` json
{
  "InstanceIds": [
    "i-04e8cc881de926954"
  ]
}
```

Execution Role:

``` text
instance_Sch
```

Click **Create schedule**.

------------------------------------------------------------------------

# Final Result

  Schedule          API              Time
  ----------------- ---------------- --------------
  Start-EC2-Daily   StartInstances   09:00 AM IST
  Stop-EC2-Daily    StopInstances    07:00 PM IST

------------------------------------------------------------------------

# Testing

1.  Edit the cron to run 2--3 minutes from the current time.
2.  Stop (or start) the EC2 instance manually.
3.  Wait for the scheduled time.
4.  Verify:
    -   Start schedule: `Stopped → Pending → Running`
    -   Stop schedule: `Running → Stopping → Stopped`
5.  Restore the production cron expressions after testing.

------------------------------------------------------------------------

# Production Best Practices

-   Use a least-privilege IAM policy.
-   Scope permissions to the specific EC2 instance ARN.
-   Keep Flexible Time Window OFF for exact execution.
-   Monitor schedule executions with CloudWatch logs/metrics if
    required.
-   Periodically review IAM permissions.

------------------------------------------------------------------------

# Troubleshooting

  -----------------------------------------------------------------------
  Issue                         Solution
  ----------------------------- -----------------------------------------
  AccessDenied                  Verify IAM role permissions

  Schedule doesn't run          Confirm cron, time zone, and schedule is
                                enabled

  Role cannot be assumed        Verify trust policy uses
                                `scheduler.amazonaws.com`

  Wrong instance affected       Verify the Instance ID in the payload
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# Success Criteria

-   Two schedules are created.
-   EC2 starts automatically at **09:00 AM IST**.
-   EC2 stops automatically at **07:00 PM IST**.
-   EventBridge Scheduler uses the `instance_Sch` execution role
    successfully.
