# AWS S3 Logs Monitoring using CloudTrail Data Events — End-to-End Guide (Console Only)

This guide sets up object-level (data event) monitoring for one or more S3 buckets using AWS CloudTrail, ships those events to CloudWatch Logs, and raises alarms on suspicious or important activity (e.g. `DeleteObject`, `PutObject`, unauthorized `GetObject`) — entirely through the AWS Management Console.

> **Cost note:** CloudTrail *management* events are free. CloudTrail *data events* (object-level S3 API calls) are billed per 100,000 events. Enabling them on high-traffic buckets can generate meaningful cost — scope them to specific buckets unless you have a reason to monitor all buckets.

---

## Prerequisites

- Console access to an AWS account with permissions for: CloudTrail, S3, CloudWatch, SNS, IAM.
- The name(s) of the S3 bucket(s) you want to monitor.

---

## Step 1 — Create a bucket to store CloudTrail logs

Use a separate bucket for logs than the bucket(s) you're monitoring, so log writes don't trigger more log writes.

1. Go to **S3 → Buckets → Create bucket**.
2. Name it something like `cloudtrail-logs-<account-id>-<region>`.
3. Leave **Block all public access** ON (default).
4. Click **Create bucket**.

You won't need to manually attach a bucket policy — when you create the trail through the console in Step 2, CloudTrail attaches the required permissions to this bucket automatically.

---

## Step 2 — Create a CloudTrail trail

1. Go to **CloudTrail → Trails → Create trail**.
2. **Trail name:** e.g. `s3-data-events-trail`.
3. **Storage location:** choose **Use existing S3 bucket** → select the bucket from Step 1.
4. Enable **Log file validation** (recommended, for integrity checking).
5. Leave **SSE-KMS encryption** off for now unless your organization requires it (see Troubleshooting → Issue 8 if you do enable it).
6. Click **Next**.
7. Under **Management events**, leave the default (Read/Write: All) unless you specifically want to exclude them.
8. Click **Next**, review, then **Create trail**.

---

## Step 3 — Enable S3 data events on the trail

This is the core step — without it, CloudTrail only logs bucket-level actions like `CreateBucket`, not object-level reads/writes.

1. Go to **CloudTrail → Trails**, click your trail name.
2. Under **Data events**, click **Edit**.
3. **Data event source:** select **S3**.
4. **Log selector template:** choose one of:
   - **Log all events** — all reads and writes,
   - **Log only write events** — recommended default for security monitoring (deletes/uploads),
   - **Log only read events**, or
   - **Custom** — build your own via advanced event selectors (e.g. to filter by `eventName` or exclude a specific bucket).
5. Under **Resource type**, choose **S3** and either:
   - Leave it scoped to **all current and future S3 buckets** (broad, higher cost), or
   - Click **Add S3 bucket** and browse to/select the specific bucket(s) you want monitored — this is the recommended scope for cost control (see the cost note at the top of this guide).
6. Click **Save changes**.

> Note: **Log only write events** is not the same as excluding read entirely at the API level — it uses CloudTrail's own read/write classification (e.g. `PutObject`/`DeleteObject` are write; `GetObject`/`ListObjects` are read). If a specific API call you care about isn't showing up, it may be classified differently than you expect — check via **Custom** with an explicit `eventName` filter instead of guessing.

---

## Step 4 — Confirm logging is active

1. Go to **CloudTrail → Trails**, click your trail.
2. Check the **Status** field — it should say **Logging**. If it shows "Stopped," click **Start logging** at the top of the page.

---

## Step 5 — Generate a test event (validation happens in Step 6b)

1. Go to **S3 → Buckets** → open the bucket you're monitoring.
2. Upload a small throwaway file (Upload → Add files → pick any small file → Upload).

> **Important:** Do **not** check CloudTrail's **Event history** page for this. Event history only ever shows *management* events — it never displays data events, no matter how long you wait. Checking there for a `PutObject` event is a dead end that looks like a failure but isn't one. The real verification happens in Step 6b once CloudWatch Logs delivery is enabled.

---

## Step 6 — Send CloudTrail events to CloudWatch Logs

Event history only retains 90 days and isn't easily alertable. Route events to CloudWatch Logs so you can build metric filters and alarms.

1. Go to **CloudTrail → Trails**, click your trail → **Edit**.
2. Scroll to the **CloudWatch Logs** section, toggle it **Enabled**.
3. **Log group:** click **New**, name it e.g. `/cloudtrail/s3-data-events`.
4. **IAM Role:** click **New**, accept the suggested role name (e.g. `CloudTrail_CloudWatchLogs_Role`) — the console will create the role with a trust policy for the CloudTrail service and a permissions policy scoped to write to this log group.
5. Click **Save changes**.

You can verify the role afterward under **IAM → Roles → CloudTrail_CloudWatchLogs_Role** — it should show a trust relationship with `cloudtrail.amazonaws.com` and a policy allowing `logs:PutLogEvents` / `logs:CreateLogStream` on your log group.

---

## Step 6b — Confirm the test event actually arrived

This is the real validation step — it replaces checking Event history.

1. Wait 10–15 minutes after uploading the test file in Step 5 (and after CloudWatch Logs delivery was enabled in Step 6).
2. Go to **CloudWatch → Log groups → `/cloudtrail/s3-data-events`**.
3. Click **Logs Insights** (or the **Search log group** button).
4. Run:
   ```
   fields eventTime, eventName, eventSource, requestParameters.bucketName
   | filter eventSource = "s3.amazonaws.com"
   | sort eventTime desc
   | limit 20
   ```
5. Confirm you see a `PutObject` entry for your test file with a recent timestamp.

If nothing appears after 15–20 minutes, see Troubleshooting → Issue 1.

---

## Step 7 — Create a metric filter on the log group

This turns raw log lines into a numeric CloudWatch metric you can alarm on. Example: alert on every `DeleteObject` call.

1. Go to **CloudWatch → Log groups**, click `/cloudtrail/s3-data-events`.
2. Go to the **Metric filters** tab → **Create metric filter**.
3. **Filter pattern**, paste:
   ```
   { ($.eventSource = "s3.amazonaws.com") && ($.eventName = "DeleteObject") }
   ```
4. (Optional) Click **Test pattern** and select a recent log event to confirm it matches before saving.
5. Click **Next**.
6. **Filter name:** `S3DeleteObjectFilter`.
7. **Metric namespace:** `S3Monitoring`.
8. **Metric name:** `S3DeleteObjectCount`.
9. **Metric value:** `1`, **Default value:** `0`.
10. Click **Next**, review, then **Create metric filter**.

Repeat this step for other event names you care about, e.g. `PutObject`, `PutBucketPolicy`, or unauthorized `GetObject` attempts (`errorCode` field present).

---

## Step 8 — Create an SNS topic for alerts

1. Go to **SNS → Topics → Create topic**.
2. Type: **Standard**. Name: `s3-monitoring-alerts`. Click **Create topic**.
3. On the topic page, click **Create subscription**.
4. **Protocol:** Email. **Endpoint:** your email address. Click **Create subscription**.
5. Check your inbox and click the confirmation link in the "AWS Notification - Subscription Confirmation" email — until you confirm, no notifications will arrive.

---

## Step 9 — Create a CloudWatch alarm on the metric filter

1. Go to **CloudWatch → Alarms → All alarms → Create alarm**.
2. Click **Select metric** → find **S3Monitoring** namespace → select **S3DeleteObjectCount**.
3. **Statistic:** Sum. **Period:** 5 minutes.
4. **Condition:** Greater/Equal → threshold `1`.
5. Click **Next**.
6. Under **Notification**, select the `s3-monitoring-alerts` SNS topic for the **In alarm** state.
7. Under **Missing data treatment**, choose **Treat missing data as good (not breaching)** to avoid false alarms during quiet periods.
8. Click **Next**, name the alarm (e.g. `S3-DeleteObject-Alert`), review, then **Create alarm**.

---

## Step 10 — Validate end-to-end

1. Go back to **S3**, open the monitored bucket, select the test file from Step 5, and delete it.
2. Wait 5–15 minutes for the event to flow through: CloudTrail → CloudWatch Logs → metric filter → alarm.
3. Check your email for the SNS alert.
4. Also check **CloudWatch → Alarms** — the alarm state should have moved to **In alarm** briefly, then back to **OK**.
5. If nothing arrived, work through Troubleshooting below in order.

---

## Step 11 (Optional) — Real-time alerting with EventBridge

CloudTrail can also publish events directly to EventBridge with lower latency than CloudWatch Logs metric filters — useful for near-real-time response (e.g. triggering a Lambda function automatically).

1. Go to **EventBridge → Rules → Create rule**.
2. Name the rule, e.g. `s3-delete-object-realtime`.
3. **Event bus:** default. **Rule type:** Rule with an event pattern.
4. **Event pattern**, choose **Custom pattern (JSON editor)** and paste:
   ```json
   {
     "source": ["aws.s3"],
     "detail-type": ["AWS API Call via CloudTrail"],
     "detail": {
       "eventSource": ["s3.amazonaws.com"],
       "eventName": ["DeleteObject", "PutBucketPolicy"]
     }
   }
   ```
5. **Target:** select SNS topic (`s3-monitoring-alerts`) or a Lambda function.
6. Review and **Create rule**.

---

## Troubleshooting

**Issue 1 — No data events appear in CloudWatch Logs**
- First, confirm you're not checking **CloudTrail → Event history** — that page never shows data events, by design, regardless of configuration. It isn't a symptom of misconfiguration; it's simply not what that page does. The only reliable places to see S3 data events are CloudWatch Logs (once delivery is enabled), the raw log files in your CloudTrail S3 bucket, or Athena queries against them.
- Go to **CloudTrail → Trails → your trail** and confirm **Status = Logging**, not "Stopped."
- Confirm the **Data events** section actually lists your bucket, not just management events — this is the most common real miss.
- Data events can take **up to 15 minutes** (sometimes longer) to appear; don't assume failure too early.
- Confirm the S3 action you performed actually matches your Read/Write setting (e.g. if you only enabled **Write**, a download/`GetObject` won't show up).

**Issue 2 — Trail creation fails or shows a bucket permission error**
- This usually only happens if you're reusing a bucket that already has a restrictive custom policy attached. Go to **S3 → your log bucket → Permissions → Bucket policy** and confirm it includes a statement allowing the CloudTrail service principal (`cloudtrail.amazonaws.com`) `s3:GetBucketAcl` and `s3:PutObject`. If you created the trail through the console against a fresh bucket, this is normally added automatically — re-check by editing the trail and re-selecting the bucket.

**Issue 3 — CloudWatch Logs log group is empty even though the trail shows "Logging"**
- Go to **CloudTrail → Trails → your trail → Edit** and confirm the CloudWatch Logs toggle is still **Enabled** and pointed at the correct log group.
- Go to **IAM → Roles → CloudTrail_CloudWatchLogs_Role** and confirm its permissions policy references the exact log group ARN (including the trailing `:*`), and its trust policy allows `cloudtrail.amazonaws.com`.
- Check you're viewing the **correct region** in the CloudWatch console — for a multi-region trail, all events are delivered to one log group in the trail's **home region** only, so switch the console region selector if you're not seeing anything.

**Issue 4 — Metric filter shows 0 even though events are landing in the log group**
- Open the log group, go to **Logs Insights**, run a quick query like `fields eventName, eventSource | limit 20` to confirm the field names actually match what your filter pattern expects (e.g. `$.eventName`, not something nested differently).
- Metric filters only evaluate **new** log events created after the filter itself was created — they don't retroactively scan existing entries. Generate a fresh test event after creating the filter, not before.

**Issue 5 — Alarm stays in INSUFFICIENT_DATA**
- This is expected until the metric actually receives a data point — it's not necessarily a misconfiguration. Trigger a real event (Step 10) and wait.
- Double-check the alarm's metric **namespace** and **name** exactly match what you set in the metric filter (Step 7) — these are case-sensitive and a common mismatch.

**Issue 6 — SNS email subscription never receives a notification**
- Go to **SNS → Subscriptions** and confirm the status is **Confirmed**, not "Pending confirmation" — unconfirmed subscriptions silently receive nothing.
- On the alarm, check the **Actions** tab shows the SNS topic listed under the **In alarm** state specifically, not only "OK" or "Insufficient data."

**Issue 7 — Costs higher than expected**
- Data events are billed per event, and read-heavy buckets can generate very high volumes fast. Go back to Step 3 and confirm you scoped data events to specific buckets rather than "all current and future buckets," and prefer logging **Write** only unless you specifically need read auditing.
- Consider CloudTrail Insights or aggregated events (rolls high-volume data events into 5-minute summaries) if you don't need per-event granularity.

**Issue 8 — KMS-encrypted trail: CloudWatch Logs or S3 delivery fails**
- If you enabled SSE-KMS on the CloudTrail log bucket, go to **KMS → Customer managed keys → your key → Key policy** and confirm both the CloudTrail service and the `CloudTrail_CloudWatchLogs_Role` have `kms:GenerateDataKey` and `kms:Decrypt` permissions, in addition to the S3/CloudWatch permissions from earlier steps.

---

## Quick reference — order of operations

1. Create log-destination S3 bucket
2. Create CloudTrail trail
3. Enable S3 data events on the trail (scope to target bucket)
4. Confirm trail status = Logging
5. Generate a test event (upload a test file)
6. Enable CloudWatch Logs delivery (log group + IAM role, via console)
6b. Verify the test event arrived, using CloudWatch Logs Insights — **not** Event history
7. Create metric filter(s) on the log group
8. Create SNS topic + subscribe + confirm
9. Create CloudWatch alarm on the metric filter, action → SNS topic
10. Validate end-to-end with a real test event
11. (Optional) Add EventBridge rule for lower-latency/automated response
