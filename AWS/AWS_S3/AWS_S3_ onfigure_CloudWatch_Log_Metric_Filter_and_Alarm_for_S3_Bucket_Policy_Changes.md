# End-to-End Practical: Configure CloudWatch Log Metric Filter and Alarm for S3 Bucket Policy Changes

# Objective

Detect whenever an **Amazon S3 Bucket Policy** is created, modified, or deleted and receive an alert using **Amazon CloudWatch Alarm + Amazon SNS**.

This is an AWS Security Best Practice and is recommended by **AWS Security Hub** and **CIS AWS Foundations Benchmark**.

---

# Architecture

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/aade6345-91bc-4d0d-bb08-a8223382edb6" />


---

# Services Used

| Service          | Purpose                      |
| ---------------- | ---------------------------- |
| CloudTrail       | Capture API calls            |
| CloudWatch Logs  | Store CloudTrail events      |
| Metric Filter    | Detect bucket policy changes |
| CloudWatch Alarm | Trigger alert                |
| SNS              | Send notification email      |

---

# Prerequisites

* AWS Account
* IAM Admin permissions
* At least one S3 bucket
* CloudTrail enabled

---

# Step 1: Create an S3 Bucket

Go to:

```
AWS Console
→ S3
→ Create Bucket
```

Example:

```
demo-secure-bucket
```

Click **Create Bucket**.

---

# Step 2: Enable CloudTrail

Go to

```
CloudTrail
→ Trails
→ Create Trail
```

Trail Name

```
organization-trail
```

Storage Location

```
Create New S3 Bucket
```

Enable

```
✔ Multi-region Trail
```

Click Next.

---

## Event Type

Choose

```
Management Events

Read

Write

✔ Enabled
```

Finish.

CloudTrail now records API activity.

---

# Step 3: Send CloudTrail Logs to CloudWatch

Open the trail.

Click

```
Edit
```

Enable

```
CloudWatch Logs
```

Create New Log Group

```
cloudtrail-log-group
```

IAM Role

```
Create New Role
```

Save.

CloudTrail now forwards events to CloudWatch Logs.

---

# Step 4: Verify Log Group

Open

```
CloudWatch

Logs

Log Groups
```

You should see

```
cloudtrail-log-group
```

---

# Step 5: Create Metric Filter

Open

```
CloudWatch

Log Groups

cloudtrail-log-group

Metric Filters

Create Metric Filter
```

---

## Filter Pattern

Paste:

```text
{ ($.eventName = PutBucketPolicy) || ($.eventName = DeleteBucketPolicy) }
```

This matches:

* PutBucketPolicy
* DeleteBucketPolicy

---

Click

```
Next
```

---

# Step 6: Configure Metric

Metric Namespace

```
Security
```

Metric Name

```
S3BucketPolicyChanges
```

Metric Value

```
1
```

Default Value

```
0
```

Click

```
Create Metric Filter
```

---

# Step 7: Verify Metric

Go to

```
CloudWatch

Metrics

Custom Namespace

Security
```

You should see

```
S3BucketPolicyChanges
```

---

# Step 8: Create SNS Topic

Go to

```
SNS

Topics

Create Topic
```

Type

```
Standard
```

Topic Name

```
SecurityAlerts
```

Create.

---

# Step 9: Create Subscription

Open

```
SecurityAlerts
```

Click

```
Create Subscription
```

Protocol

```
Email
```

Endpoint

```
your-email@example.com
```

Create.

---

Open your email.

Click

```
Confirm Subscription
```

## Now there is an issue here if you click on confirm subscription after some time aws or your browser automatically selects the unsubscribe and your are not able to receive the alert on your mail so after exploring i have this solution 

## Solution is instead if clicking on the confirmation subscription just do right click on confirm subscription and copy the URl and make share to chatgpt , chatgpt will convert that URL into this type of command 


```bash
Command

aws sns confirm-subscription \
  --topic-arn "<YOUR_TOPIC_ARN>" \
  --token "<YOUR_CONFIRMATION_TOKEN>" \
  --authenticate-on-unsubscribe true

Example

aws sns confirm-subscription \
  --topic-arn "arn:aws:sns:ap-south-1:123456789012:SecurityAlerts" \
  --token "2336412f37fb687f5d51e6e2425929f52bf0d75d43a807fce1be6cbfab2588f01xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  --authenticate-on-unsubscribe true

Parameters

--topic-arn : The ARN of the SNS topic.
--token : The confirmation token received in the subscription email.
--authenticate-on-unsubscribe true : Requires AWS authentication to unsubscribe, helping prevent accidental or unauthorized unsubscribe requests.

Successful Output
{
  "SubscriptionArn": "arn:aws:sns:ap-south-1:123456789012:SecurityAlerts:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}

```

---

# Step 10: Create CloudWatch Alarm

Go to

```
CloudWatch

Alarms

Create Alarm
```

Choose Metric

```
Security

S3BucketPolicyChanges
```

---

Condition

```
Greater than or equal to

1
```

Evaluation Period

```
1 Minute

1 Datapoint
```

Next.

---

# Step 11: Configure Notification

Choose

```
Existing SNS Topic

SecurityAlerts
```

Next.

Alarm Name

```
S3-Bucket-Policy-Modified
```

Create Alarm.

---

# Step 12: Test the Alarm

Go to

```
S3

demo-secure-bucket

Permissions

Bucket Policy
```

Add:

```json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Principal":"*",
      "Action":"s3:GetObject",
      "Resource":"arn:aws:s3:::demo-secure-bucket/*"
    }
  ]
}
```

Save.

---

# Step 13: What Happens?

```text
Bucket Policy Modified
          │
          ▼
CloudTrail Records API Call
          │
          ▼
CloudWatch Log Group
          │
          ▼
Metric Filter Matches
(PutBucketPolicy)
          │
          ▼
Metric Value = 1
          │
          ▼
CloudWatch Alarm
          │
          ▼
SNS Notification
          │
          ▼
Email Sent
```

---

# Step 14: Check Email

You should receive an email similar to:

```
ALARM: S3-Bucket-Policy-Modified

Metric: S3BucketPolicyChanges

Threshold crossed

Current Value: 1
```

---

# Step 15: Test Delete Bucket Policy

Delete the bucket policy.

CloudTrail logs:

```
DeleteBucketPolicy
```

The metric filter matches again and triggers another alarm.

---

# Filter Pattern Explained

```text
{ ($.eventName = PutBucketPolicy) || ($.eventName = DeleteBucketPolicy) }
```

| Event              | Meaning                          |
| ------------------ | -------------------------------- |
| PutBucketPolicy    | Bucket policy created or updated |
| DeleteBucketPolicy | Bucket policy removed            |

---

# Complete End-to-End Flow

```text
                AWS Administrator
                        │
                        ▼
         Create / Modify Bucket Policy
                        │
                        ▼
                Amazon CloudTrail
                        │
                        ▼
           CloudWatch Logs Log Group
                        │
                        ▼
             CloudWatch Metric Filter
                        │
                        ▼
              Custom Security Metric
                        │
                        ▼
               CloudWatch Alarm
                        │
                        ▼
                  Amazon SNS Topic
                        │
                        ▼
              Email to Security Team
```

---

# Validation Checklist

✅ CloudTrail enabled

✅ CloudTrail integrated with CloudWatch Logs

✅ Log Group created

✅ Metric Filter created

✅ Custom Metric visible

✅ SNS Topic created

✅ Email subscription confirmed

✅ CloudWatch Alarm created

✅ Bucket policy modified

✅ Alarm triggered

✅ Email notification received

---

# Real-World Use Case

A security engineer accidentally changes an S3 bucket policy to allow public access.

Within seconds:

1. CloudTrail records the `PutBucketPolicy` API call.
2. CloudWatch Logs receives the event.
3. The metric filter detects the policy change.
4. A custom metric increments.
5. The CloudWatch Alarm enters the **ALARM** state.
6. Amazon SNS sends an email to the security team.
7. The team investigates and reverts the unauthorized policy change.

This setup provides continuous monitoring and rapid alerting for sensitive S3 bucket policy changes in production AWS environments.
