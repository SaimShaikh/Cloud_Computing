# Amazon S3 → Amazon EventBridge: 

**Use case:** You have hit the hard limit of **100 event notification configurations per S3 bucket** and cannot add another SQS / SNS / Lambda notification. This guide migrates that bucket to **Amazon EventBridge**, where routing is done by rules on the default event bus instead of by entries inside the bucket's notification sub-resource.



---



## 1. Scenario and the exact quota you hit

### 1.1 What the error looks like

When you try to add the 101st notification you get one of these:

```
An error occurred (InvalidArgument) when calling the PutBucketNotificationConfiguration operation:
Configuration exceeds the maximum number of allowed notification configurations.
```

Or, if the problem is filter overlap rather than count:

```
An error occurred (InvalidArgument) when calling the PutBucketNotificationConfiguration operation:
Configuration is ambiguously defined. Cannot have overlapping suffixes in two rules
if the prefixes are overlapping for the same event type.
```

### 1.2 The two limits that block you

| Limit | Value | Adjustable? | Where enforced |
|-------|-------|-------------|----------------|
| Event notifications per S3 bucket | **100** | **No** — hard limit, no Service Quotas request possible | S3 `PutBucketNotificationConfiguration` |
| Overlapping prefix/suffix for the same event type | Not allowed | No | S3 notification validation |

A "notification configuration" = one entry in `TopicConfigurations` + `QueueConfigurations` + `LambdaFunctionConfigurations` combined. Ten Lambda triggers + ninety SQS triggers = 100 = full.

The overlap rule is often the *real* blocker even below 100: you cannot point `raw/` → LambdaA and `raw/incoming/` → LambdaB for the same `s3:ObjectCreated:*` event type.

### 1.3 Why EventBridge removes both problems

- The bucket holds **exactly one** EventBridge setting: a boolean. It does not consume any of the 100 slots and never conflicts with itself.
- S3 sends **every** event type to the default event bus. Filtering moves out of S3 and into EventBridge rules.
- EventBridge rules **may overlap freely**. Ten rules can all match the same object and all fire.
- Rule quota is **300 per event bus by default and is adjustable to thousands**, versus a hard 100.

### 1.4 Important: EventBridge is not a "quota increase"

You are changing the delivery contract. The event JSON shape is completely different, so **every consumer's parsing code must change**. That is the real work in this migration, and Part H covers doing it without dropping events.

---

## 2. Your options (and why EventBridge wins)

| # | Option | How it works | Pros | Cons | Verdict |
|---|--------|--------------|------|------|---------|
| 1 | **EventBridge** | Bucket → default bus → rules → targets | No per-bucket limit, overlapping filters allowed, 20+ target types, SQS FIFO supported, content filtering on size/reason/requester, archive & replay, DLQ + retry per target | New event schema, 300 rules/bus soft cap, extra hop (slightly higher latency) | **Recommended** |
| 2 | **SNS fan-out** | 1 notification → 1 SNS topic → N subscribers | Uses only 1 of the 100 slots, cheap, mature | Filtering limited to SNS message-filter policies on attributes (S3 sends none by default), 12.5M subscriptions but no key-level routing, no replay | Good for simple broadcast |
| 3 | **Single dispatcher Lambda** | 1 notification → 1 Lambda → routes in code | 1 slot, unlimited routing logic | You own the routing code, retries, DLQ, scaling; becomes a single point of failure | Works, but you rebuild EventBridge badly |
| 4 | **SQS + consumer routing** | 1 notification → 1 queue → workers decide | 1 slot, buffered, replayable via redrive | Same as above; polling cost | Fine for batch pipelines |
| 5 | **Split the bucket** | Move prefixes to new buckets | Resets the 100 per new bucket | Data migration, path changes, app rewrites, IAM churn | Last resort |
| 6 | **CloudTrail data events → EventBridge** | S3 data events into CloudTrail, matched via `AWS API Call via CloudTrail` | Captures reads (`GetObject`) too | Costs per data event, higher latency, best-effort delivery | Only when you need read events |

> **Note on option 6:** the modern S3→EventBridge integration is **direct** and does **not** require CloudTrail. Only enable CloudTrail data events if you need events EventBridge does not emit natively (e.g. `GetObject`, `HeadObject`).

You can combine: EventBridge rule → SNS topic → many subscribers is a valid way to stay under 300 rules.

---

## 3. S3 Event Notifications vs EventBridge — full comparison

| Dimension | S3 Event Notifications | S3 → EventBridge |
|-----------|------------------------|------------------|
| Config location | Bucket notification sub-resource | Bucket boolean + EventBridge rules |
| Per-bucket limit | 100 configurations (hard) | 1 boolean; 300 rules/bus (adjustable) |
| Overlapping filters | Not allowed for same event type | Allowed, unrestricted |
| Selecting event types | You choose each type | All types are sent; you filter in rules |
| Destinations | SQS (standard only), SNS, Lambda, EventBridge | 20+ targets incl. Lambda, SQS standard **and FIFO**, SNS, Step Functions, Kinesis Data Streams, Firehose, ECS task, Batch, SSM Run Command/Automation, CodeBuild, CodePipeline, SageMaker Pipeline, Redshift Data API, Glue workflow, Inspector, API Destination (any HTTPS), another event bus, CloudWatch Logs |
| Filtering power | Prefix + suffix on key only | Prefix, suffix, wildcard, anything-but, numeric, exists, cidr, case-insensitive, `$or` — on key, size, etag, reason, requester, source IP, storage class, tier, version-id, deletion-type |
| Filter on object size | No | Yes (`detail.object.size` numeric matcher) |
| Filter on who/how | No | Yes (`requester`, `source-ip-address`, `reason`) |
| Cross-account | Via SNS/SQS policies | Native: forward to another account's event bus |
| Cross-region | No | Yes: bus-to-bus forwarding |
| Delivery semantics | At least once | At least once |
| Ordering | Not guaranteed | Not guaranteed |
| Typical latency | Seconds (can exceed a minute) | Seconds (can exceed a minute) |
| Retry / DLQ | Depends on destination | Per-target retry policy + DLQ |
| Replay of past events | No | Yes (archive + replay) |
| Test event on setup | `s3:TestEvent` to SQS/SNS | **None** — no test event is sent |
| Object key encoding | **URL-encoded** (space → `+`) | **Not URL-encoded** |
| Payload envelope | `Records[]` array | Flat CloudWatch-Events envelope with `detail` |
| Schema registry support | No | Yes |
| Cost of the plumbing | Free | Free for AWS-service events on the default bus |
| Directory buckets (S3 Express One Zone) | Not supported | Not supported |

---

## 4. Architecture

### 4.1 Before — what you have now (and why it is full)

```
                       ┌──────────────────────────────────────┐
                       │  S3 bucket notification sub-resource │
                       │  (HARD LIMIT: 100 entries)           │
                       ├──────────────────────────────────────┤
   PutObject ─────────►│  #1   raw/        .csv  → LambdaA    │
                       │  #2   raw/        .json → SQS-1      │
                       │  #3   images/     .jpg  → SNS-1      │
                       │  ...                                 │
                       │  #100 archive/    .zip  → LambdaZ    │
                       │  #101 ────────► ✗ REJECTED           │
                       └──────────────────────────────────────┘
```

### 4.2 After — EventBridge

```
                    ┌───────────────────────────┐
                    │  S3 bucket                │
                    │  EventBridge = ENABLED    │  ← one boolean, zero slots used
                    └─────────────┬─────────────┘
                                  │ all 11 event types, no filtering here
                                  ▼
                    ┌───────────────────────────┐
                    │  EventBridge DEFAULT bus  │  (ap-south-1, same region as bucket)
                    │  source = "aws.s3"        │
                    └─────────────┬─────────────┘
                                  │  event patterns evaluated in parallel
        ┌─────────────────┬───────┴────────┬──────────────────┬─────────────────┐
        ▼                 ▼                ▼                  ▼                 ▼
 ┌─────────────┐  ┌──────────────┐  ┌─────────────┐   ┌──────────────┐  ┌──────────────┐
 │ Rule: raw/  │  │ Rule: images/│  │ Rule:       │   │ Rule: large  │  │ Rule: catch  │
 │ *.csv       │  │ *.jpg        │  │ Object      │   │ objects      │  │ all (debug)  │
 │ Created     │  │ Created      │  │ Deleted     │   │ size > 5GB   │  │              │
 └──────┬──────┘  └──────┬───────┘  └──────┬──────┘   └──────┬───────┘  └──────┬───────┘
        │                │                 │                 │                 │
        ▼                ▼                 ▼                 ▼                 ▼
    Lambda-A      Step Functions       SQS FIFO         SNS topic      CloudWatch Logs
        │                                  │                 │
        └── DLQ (SQS) ◄────────────────────┴─────────────────┘
            retry: 2 attempts / 1 h max age
```

**Key structural points**

1. S3 always publishes to the **default** event bus of the bucket's region. You cannot point S3 at a custom bus directly — forward with a rule if you need one.
2. Overlapping rules all fire. In the diagram, a `raw/report.csv` upload larger than 5 GB triggers the `raw/*.csv` rule, the `large objects` rule, **and** the catch-all.
3. Each rule may have up to **5 targets**. Need more? Fan out via SNS, or add another rule with the same pattern.

---

## 5. Prerequisites

| Requirement | Detail |
|-------------|--------|
| AWS account | With permission to modify the bucket and create EventBridge rules |
| Region | All resources in the **same region as the bucket** |
| IAM permissions | See Part J — minimum set listed there |
| Existing bucket | Or create one in Part B |
| Time | ~60–90 minutes for the full lab |
| Cost | Under ₹10 / $0.15 if you follow the cleanup section |

Confirm your account and region before starting: click your account name in the top-right corner of the AWS Console to see your Account ID, and check the region selector next to it for your current working region.

---

## 6. Lab values

Use these placeholder values throughout the console steps in this guide — swap in your own bucket name, region, and account ID as you go.

| Placeholder | Meaning |
|-------------|---------|
| `ap-south-1` | Region of the bucket **and** of every rule/target |
| `111122223333` | Your AWS Account ID — shown top-right in the console |
| `ebridge-lab-ap-south-1-111122223333` | The bucket being migrated |
| `default` event bus | S3 cannot publish anywhere else |

---

## 7. Part A — Pre-flight: audit the existing notification configuration

**Do not skip this.** S3's notification configuration is a single document behind the scenes, and some tools (scripts, SDKs, infrastructure-as-code) replace it wholesale rather than patching it — so it is worth knowing exactly what is configured before you touch anything. Working from the console, as this guide does, adds and removes one notification at a time and is safe by construction; this step is still worth doing so you have a record to fall back on.

### A.1 List every existing notification

1. Console → **S3** → your bucket → **Properties** tab
2. Scroll to **Event notifications**
3. Below the EventBridge panel you'll see the list of individual notifications — each row shows its name, event types, prefix/suffix filter, and destination
4. Copy this list into a table (spreadsheet or notes) before changing anything — you will need it for the migration matrix in Part H

### A.2 Count how many slots are used

Count the rows in that same list. If you're at 100, S3's console will refuse to let you add another — that refusal is the quota you're migrating away from.

### A.3 Check whether EventBridge is already on

Still on the **Event notifications** page, look at the **Amazon EventBridge** panel above the list. **On** means S3 is already publishing to the bus and you only need rules; **Off** means you'll enable it in Part B.

### A.4 Build the migration matrix

Using the list from A.1, fill in a table like this before touching anything:

| Old notification name | Event types | Prefix | Suffix | Destination | New EventBridge rule name | Owner |
|------------------------|-------------|--------|--------|-------------|---------------------------|-------|
| `csv-ingest` | Object creation events | `raw/` | `.csv` | LambdaA | `s3-raw-csv-created` | data-eng |
| `img-thumb` | Object creation events (Put) | `images/` | `.jpg` | SQS-1 | `s3-images-jpg-created` | media |

### A.5 Record the bucket's region (rules must live in the same region)

The bucket's Region is shown on the **Properties** tab, under **Bucket overview**, and also next to the bucket name on the main S3 bucket list.

---

## 8. Part B — Console walkthrough (enable + verify)

This part is entirely console-based.

### B.1 (Optional) Create a lab bucket

1. Console → **S3** → **Buckets** → **Create bucket**
2. **Bucket type:** General purpose *(directory buckets do not support notifications at all)*
3. **Bucket name:** `ebridge-lab-ap-south-1-111122223333`
4. **Region:** Asia Pacific (Mumbai) ap-south-1
5. **Block Public Access:** leave all four boxes checked
6. **Bucket Versioning:** **Enable** — needed later to see `Delete Marker Created` vs `Permanently Deleted`
7. Leave encryption at the default SSE-S3
8. **Create bucket**

### B.2 Enable EventBridge on the bucket

1. Open the bucket → **Properties** tab
2. Scroll to **Event notifications**
3. Find the **Amazon EventBridge** panel (it sits *above* the individual event notification list)
4. Click **Edit**
5. **Send notifications to Amazon EventBridge for all events in this bucket** → **On**
6. **Save changes**

**What just happened:** S3 added `EventBridgeConfiguration: {}` to the bucket's notification sub-resource. Your existing 100 notifications are untouched — the console does a read-modify-write for you. It is only a raw, hand-built API call (SDK, script, or infrastructure-as-code) that is destructive.

> ⏱ **Wait ~5 minutes.** AWS documents that it can take around five minutes for events to begin flowing after you enable this. Testing immediately and seeing nothing is the single most common false alarm in this migration.

### B.3 Create a catch-all debug rule → CloudWatch Logs

This rule exists so you can *see* the raw events before you wire real targets. It is temporary — delete it in Part R.

1. Console → **Amazon EventBridge** → **Buses** → **Rules** → make sure **Event bus = default**
2. **Create rule**
3. **Name:** `s3-catchall-debug`
4. **Description:** `TEMPORARY - logs every S3 event from the lab bucket`
5. **Event bus:** `default`
6. Leave **Enable the rule on the selected event bus** checked
7. **Rule type:** **Rule with an event pattern** → **Next**
8. **Event source:** **AWS events or EventBridge partner events**
9. Skip the **Sample event** box (optional, but handy — pick **AWS events** → `S3 Object Created`)
10. **Creation method:** **Custom pattern (JSON editor)**
11. Paste:

```json
{
  "source": ["aws.s3"],
  "detail": {
    "bucket": {
      "name": ["ebridge-lab-ap-south-1-111122223333"]
    }
  }
}
```

12. **Next**
13. **Target types:** **AWS service** → **Target:** **CloudWatch log group**
14. **Log Group:** type `/aws/events/s3-catchall` — the console creates it
15. **Next** → skip tags → **Next** → **Create rule**

> The console silently creates the CloudWatch Logs **resource policy** that lets `events.amazonaws.com` write to the log group. If you create this target outside the console (SDK, IaC) you must create that policy yourself. Missing it is the #1 cause of "rule fires but log group stays empty".

### B.4 Generate a test event

1. Go back to **S3** → your bucket → **Upload**
2. Upload any small file, e.g. `hello.txt`
3. Return to **CloudWatch** → **Log groups** → `/aws/events/s3-catchall` → newest log stream

You should see an event like:

```json
{
  "version": "0",
  "id": "17793124-05d4-b198-2fde-7ededc63b103",
  "detail-type": "Object Created",
  "source": "aws.s3",
  "account": "111122223333",
  "time": "2026-09-17T09:12:41Z",
  "region": "ap-south-1",
  "resources": ["arn:aws:s3:::ebridge-lab-ap-south-1-111122223333"],
  "detail": {
    "version": "0",
    "bucket": { "name": "ebridge-lab-ap-south-1-111122223333" },
    "object": {
      "key": "hello.txt",
      "size": 13,
      "etag": "9a0364b9e99bb480dd25e1f0284c8555",
      "version-id": "3HL4kqtJlcpXroDTDmjVBH40Nrjfkd",
      "sequencer": "0062E99A88DC407460"
    },
    "request-id": "N4N7GDK58NMKJ12R",
    "requester": "111122223333",
    "source-ip-address": "203.0.113.42",
    "reason": "PutObject"
  }
}
```

**Read this envelope carefully — it is the whole migration.** Compare to the old format:

| What you want | Old S3 notification | EventBridge |
|---------------|---------------------|-------------|
| Bucket name | `Records[0].s3.bucket.name` | `detail.bucket.name` |
| Object key | `Records[0].s3.object.key` *(URL-encoded)* | `detail.object.key` *(not encoded)* |
| Object size | `Records[0].s3.object.size` | `detail.object.size` |
| ETag | `Records[0].s3.object.eTag` | `detail.object.etag` |
| Version id | `Records[0].s3.object.versionId` | `detail.object.version-id` |
| Event name | `Records[0].eventName` = `ObjectCreated:Put` | `detail-type` = `Object Created` + `detail.reason` = `PutObject` |
| Event time | `Records[0].eventTime` | `time` (top level) |
| Caller IP | `Records[0].requestParameters.sourceIPAddress` | `detail.source-ip-address` |
| Batching | Array of `Records` | **One event per message, never batched** |

### B.5 Confirm the rule is matching (metrics)

1. EventBridge → **Rules** → `s3-catchall-debug` → **Monitoring** tab
2. Look for **MatchedEvents** > 0 and **Invocations** > 0
3. If `MatchedEvents` is 0 → the **pattern** is wrong (Part J, row 1)
4. If `MatchedEvents` > 0 but `Invocations` is 0 or `FailedInvocations` > 0 → the **target permission** is wrong (Part J, row 4)

That two-metric split is the fastest diagnostic in the whole service. Memorise it.

---

## 9. Part C — Targets

Each rule supports **up to 5 targets**, and that number cannot be increased. Every target below is shown console-first with the exact permission it needs.

### C.1 Target: AWS Lambda

**Console**

1. EventBridge → **Create rule** → name `s3-raw-csv-to-lambda`
2. Event pattern:

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": { "name": ["ebridge-lab-ap-south-1-111122223333"] },
    "object": { "key": [{ "wildcard": "raw/*.csv" }] }
  }
}
```

3. **Target:** AWS service → **Lambda function** → pick your function
4. Create rule

**Permission model:** EventBridge invokes Lambda using a **resource-based policy on the function** — there is no IAM role involved. The console adds it automatically; outside the console you must add it yourself.

**Handler code — the part that actually changes**

```python
import json
import urllib.parse

def lambda_handler(event, context):
    # EventBridge delivers ONE event, not a Records[] array
    detail = event["detail"]
    bucket = detail["bucket"]["name"]

    # NOT URL-encoded here. The old S3 notification format WAS encoded.
    key = detail["object"]["key"]

    size = detail["object"].get("size")          # absent on some event types
    reason = detail.get("reason")                # PutObject | CompleteMultipartUpload | CopyObject | POST Object
    version_id = detail["object"].get("version-id")
    sequencer = detail["object"].get("sequencer")

    print(json.dumps({
        "bucket": bucket, "key": key, "size": size,
        "reason": reason, "versionId": version_id, "sequencer": sequencer,
        "eventId": event["id"], "eventTime": event["time"]
    }))
    return {"ok": True}
```

> **Migration trap:** old S3-notification handlers almost always contain
> `key = urllib.parse.unquote_plus(record['s3']['object']['key'])`.
> Applying `unquote_plus` to an EventBridge key **corrupts** any key containing a literal `+` or `%`. Remove the unquote, do not keep it "just in case".

**Compatibility shim** — if you cannot change consumers yet, wrap the old handler:

```python
def lambda_handler(event, context):
    d = event["detail"]
    legacy = {"Records": [{
        "eventVersion": "2.1",
        "eventSource": "aws:s3",
        "awsRegion": event["region"],
        "eventTime": event["time"],
        "eventName": "ObjectCreated:Put",
        "s3": {
            "bucket": {"name": d["bucket"]["name"]},
            "object": {
                "key": urllib.parse.quote_plus(d["object"]["key"]),
                "size": d["object"].get("size"),
                "eTag": d["object"].get("etag"),
                "versionId": d["object"].get("version-id"),
                "sequencer": d["object"].get("sequencer"),
            }
        }
    }]}
    return original_handler(legacy, context)
```

Use the shim to cut over fast, then delete it. It is technical debt, not a design.

### C.2 Target: Amazon SQS (standard)

1. Create rule `s3-created-to-sqs`, same pattern style as above
2. **Target:** AWS service → **SQS queue** → select queue
3. Optionally set **Message group ID** (FIFO only — see C.3)

**Permission model:** a **queue policy** (resource-based), not a role.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowEventBridgeRule",
    "Effect": "Allow",
    "Principal": { "Service": "events.amazonaws.com" },
    "Action": "sqs:SendMessage",
    "Resource": "arn:aws:sqs:ap-south-1:111122223333:s3-events-queue",
    "Condition": {
      "ArnEquals": { "aws:SourceArn": "arn:aws:events:ap-south-1:111122223333:rule/s3-created-to-sqs" }
    }
  }]
}
```

> Always scope with `aws:SourceArn`. A policy that allows `events.amazonaws.com` with no condition lets **any** EventBridge rule in **any** account write to your queue — the confused-deputy problem.

> If the queue uses **SSE-KMS with a customer managed key**, you must also add `kms:GenerateDataKey*` and `kms:Decrypt` for `events.amazonaws.com` to the **key policy**. SSE-SQS (AWS managed) works with no extra step.

### C.3 Target: Amazon SQS **FIFO** — impossible with plain S3 notifications

This alone justifies EventBridge for some teams. S3 Event Notifications cannot target a FIFO queue; EventBridge can.

1. Create the FIFO queue (name must end `.fifo`), enable **content-based deduplication** or plan to set a dedup ID
2. Create the rule, choose the FIFO queue as target
3. **Message group ID** is **required** — the console shows the field once you pick a `.fifo` queue

Choosing the group ID:

| Group ID value | Effect |
|----------------|--------|
| A constant, e.g. `s3-events` | Total ordering, but throughput limited to one consumer at a time |
| The object prefix/tenant | Ordering per tenant, parallelism across tenants — **usually correct** |
| The object key | Ordering per object, maximum parallelism |

> FIFO gives you ordered *delivery*, but S3 itself does not guarantee the *order events are generated*. Two rapid overwrites of one key can still arrive out of order. Compare `detail.object.sequencer` lexicographically (longer string wins; pad to equal length) to decide which event is newer.

### C.4 Target: Amazon SNS

1. Rule → **Target:** SNS topic
2. Add a topic policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowEventBridgePublish",
    "Effect": "Allow",
    "Principal": { "Service": "events.amazonaws.com" },
    "Action": "SNS:Publish",
    "Resource": "arn:aws:sns:ap-south-1:111122223333:s3-events-topic",
    "Condition": {
      "ArnEquals": { "aws:SourceArn": "arn:aws:events:ap-south-1:111122223333:rule/s3-created-to-sns" }
    }
  }]
}
```

Use this as your **fan-out escape hatch**: one rule → one topic → many subscribers keeps you far below the 300-rule ceiling.

### C.5 Target: AWS Step Functions

Unlike Lambda/SQS/SNS, Step Functions requires an **IAM role that EventBridge assumes**.

Trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "events.amazonaws.com" },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": { "aws:SourceAccount": "111122223333" },
      "ArnLike": { "aws:SourceArn": "arn:aws:events:ap-south-1:111122223333:rule/*" }
    }
  }]
}
```

Permissions policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "states:StartExecution",
    "Resource": "arn:aws:states:ap-south-1:111122223333:stateMachine:ProcessObject"
  }]
}
```

**Rule of thumb:** Lambda, SQS, SNS, and CloudWatch Logs use **resource policies**. Everything else (Step Functions, Kinesis, Firehose, ECS, Batch, SSM, CodeBuild, CodePipeline, API Destinations, another event bus) uses a **role** that you attach to the target.

### C.6 Target: Kinesis Data Streams / Firehose

Needs a role with `kinesis:PutRecord` + `kinesis:PutRecords` (or `firehose:PutRecord*`). For Kinesis you may set a **partition key path**, e.g. `$.detail.object.key`, so all events for one object land on one shard.

### C.7 Target: API Destination (any HTTPS endpoint)

1. EventBridge → **API destinations** → **Create**
2. Create a **Connection** (API key / Basic / OAuth) — credentials are stored in Secrets Manager automatically
3. Set **Invocation rate limit** (this is your backpressure control against a slow partner)
4. Use the API destination as a rule target with a role granting `events:InvokeApiDestination`

This replaces the "S3 → SNS → HTTPS subscription" pattern with retries, auth handling and rate limiting built in.

### C.8 Target: another event bus (cross-account / cross-region)

Pattern: `S3 (account A, ap-south-1) → default bus A → rule → default bus in account B / region us-east-1 → rules there`.

- Target bus needs a **resource-based policy** allowing `events:PutEvents` from account A
- Source rule needs a role with `events:PutEvents` on the target bus ARN
- Events forwarded this way are **billed as custom events**
- Only forward what you need — never forward a catch-all across regions

### C.9 Target: CloudWatch Logs

Covered in B.3. Two requirements the console handles for you but that you must set up yourself outside the console:

- The target ARN must end with `:*` → `arn:aws:logs:ap-south-1:111122223333:log-group:/aws/events/s3-catchall:*`
- A log-group resource policy is mandatory

> **Cost warning:** a catch-all → CloudWatch Logs rule on a busy production bucket ingests every event at CloudWatch Logs ingestion rates. On a bucket doing millions of PUTs a day this is the most expensive thing in the entire design. Use it to debug, then delete it.

---

## 10. Part D — Reliability: retries, DLQ, input transformer, archive & replay

### D.1 Retry policy (per target)

| Setting | Default | Range | Meaning |
|---------|---------|-------|---------|
| **Maximum age of event** | 24 hours (86400 s) | 60 – 86400 s | Stop retrying after this age |
| **Retry attempts** | 185 | 0 – 185 | Hard cap on attempts |

EventBridge retries with exponential backoff. Whichever limit is hit first ends the attempt; the event then goes to the DLQ if configured, otherwise it is **dropped silently**.

Console: rule → **Targets** → expand **Additional settings** → **Retry policy**.

Practical values:

- Interactive / near-real-time path: max age **3600 s**, attempts **5**
- Batch path where late is better than lost: max age **86400 s**, attempts **185**

### D.2 Dead-letter queue (per target)

Always attach one for production targets. Without a DLQ, an event that exhausts retries is gone with no record.

Queue policy for the DLQ:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowEventBridgeDLQ",
    "Effect": "Allow",
    "Principal": { "Service": "events.amazonaws.com" },
    "Action": "sqs:SendMessage",
    "Resource": "arn:aws:sqs:ap-south-1:111122223333:s3-events-dlq",
    "Condition": {
      "ArnEquals": { "aws:SourceArn": "arn:aws:events:ap-south-1:111122223333:rule/s3-raw-csv-to-lambda" }
    }
  }]
}
```

DLQ messages carry **message attributes** that tell you what failed:

| Attribute | Contents |
|-----------|----------|
| `RULE_ARN` | Which rule |
| `TARGET_ARN` | Which target |
| `ERROR_CODE` | e.g. `NO_PERMISSIONS`, `TARGET_NOT_FOUND`, `EVENTS_IN_BATCH_REQUEST_REJECTED` |
| `ERROR_MESSAGE` | Human-readable reason |
| `EXHAUSTED_RETRY_CONDITION` | `MaximumRetryAttempts` or `MaximumEventAgeInSeconds` |

The body is the original event, so redrive is just "read from DLQ and re-invoke the target".

> Alarm on `DeadLetterInvocations` and on `InvocationsFailedToBeSentToDlq`. The second one means you lost the event entirely — usually a missing DLQ queue policy.

### D.3 Input transformer

By default the target receives the **whole event**. The input transformer reshapes it, which is useful when the target is a legacy API, Step Functions, or an SNS email.

Input paths map:

```json
{
  "bucket": "$.detail.bucket.name",
  "key": "$.detail.object.key",
  "size": "$.detail.object.size",
  "time": "$.time"
}
```

Input template (JSON output):

```json
{"bucket":"<bucket>","key":"<key>","bytes":<size>,"occurredAt":"<time>"}
```

Input template (plain text, e.g. for an SNS email):

```
New object <key> (<size> bytes) landed in <bucket> at <time>.
```

Rules:

- Unquoted `<size>` yields a **number**; quoted `"<size>"` yields a **string**
- Up to **100** input path entries
- Referencing a path that is absent in a given event produces an empty value — guard with an `exists` filter in the pattern instead of hoping
- `$.detail.object.key` — remember EventBridge keys are **not** URL-encoded

Alternatives to the transformer:

| Setting | Behaviour |
|---------|-----------|
| **Matched event** (default) | Full event JSON |
| **Part of the matched event** (`InputPath`) | A single JSONPath extract, e.g. `$.detail` |
| **Constant** (`Input`) | A fixed JSON literal, event discarded |
| **Input transformer** | Template built from extracted paths |

### D.4 Archive and replay — your migration safety net

An **archive** stores every event matching a pattern on the bus, for a retention period you choose. **Replay** re-sends archived events to selected rules over a chosen time window.

Why this matters for your quota migration: you can archive S3 events **before** the consumers are ready, then replay them once they are. Nothing is lost during cutover.

Console: EventBridge → **Archives** → **Create archive** → bus `default` → retention (0 = indefinite) → optional pattern → create.

Then: **Replays** → **Start new replay** → choose archive, start/end time, destination bus and **specific rules**.

Behaviour to know:

- Replayed events carry a `replay-name` field in `detail` — make consumers idempotent or skip replays explicitly
- Replay targets **existing** rules; delete a rule and its archived events have nowhere to go
- Replay is **not** ordered
- Archive storage is billed per GB-month; replay is billed per GB processed

---

## 11. Part E — Event pattern cookbook (all matching possibilities)

### E.0 The three rules that govern every pattern

1. **Every value is an array.** `"name": ["a"]` — never `"name": "a"`.
2. **Values inside one array are OR'ed. Different fields are AND'ed.** So `"key": [{"prefix":"raw/"},{"suffix":".csv"}]` means *prefix raw/ **OR** suffix .csv*, not AND. To AND two conditions on the **same** field, use `wildcard` or `$or`.
3. **The pattern must mirror the event's structure.** You cannot match `detail.object.key` by writing `"key": [...]` at the top level.

### E.1 Match absolutely everything from S3 (whole account)

```json
{ "source": ["aws.s3"] }
```

### E.2 One bucket, all event types

```json
{
  "source": ["aws.s3"],
  "detail": { "bucket": { "name": ["my-bucket"] } }
}
```

### E.3 Several buckets

```json
{
  "source": ["aws.s3"],
  "detail": { "bucket": { "name": ["bucket-a", "bucket-b", "bucket-c"] } }
}
```

### E.4 All buckets sharing a naming convention

```json
{
  "source": ["aws.s3"],
  "detail": { "bucket": { "name": [{ "prefix": "prod-landing-" }] } }
}
```

This is the multi-bucket pattern that S3 notifications can never express — one rule covering 200 buckets.

### E.5 Only object creations

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": { "bucket": { "name": ["my-bucket"] } }
}
```

### E.6 Prefix only (the old `prefix` filter)

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": { "name": ["my-bucket"] },
    "object": { "key": [{ "prefix": "raw/incoming/" }] }
  }
}
```

### E.7 Suffix only (the old `suffix` filter)

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": { "object": { "key": [{ "suffix": ".csv" }] } }
}
```

### E.8 Prefix **AND** suffix — use `wildcard`, not two matchers

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": { "name": ["my-bucket"] },
    "object": { "key": [{ "wildcard": "raw/incoming/*.csv" }] }
  }
}
```

The equivalent with `$or`, if you prefer explicitness:

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "$or": [
    { "detail": { "object": { "key": [{ "wildcard": "raw/*.csv" }] } } },
    { "detail": { "object": { "key": [{ "wildcard": "raw/*.CSV" }] } } }
  ]
}
```

### E.9 Wildcard with multiple segments (date-partitioned lakes)

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "object": { "key": [{ "wildcard": "tenant-*/year=2026/month=*/*.parquet" }] }
  }
}
```

`*` matches any sequence including `/`. There is no single-character wildcard, and a pattern may contain at most 5 `*` per value. Escape a literal asterisk as `\\*`.

### E.10 Case-insensitive matching

```json
{
  "detail": {
    "object": {
      "key": [
        { "suffix": { "equals-ignore-case": ".CSV" } }
      ]
    }
  }
}
```

Also valid: `{"prefix": {"equals-ignore-case": "RAW/"}}` and `{"equals-ignore-case": "exact-name.txt"}`.

### E.11 Exclusions — `anything-but`

Exclude specific values:

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": { "object": { "key": [{ "anything-but": ["index.html", "favicon.ico"] }] } }
}
```

Exclude a prefix (ignore temp/staging writes):

```json
{
  "detail": { "object": { "key": [{ "anything-but": { "prefix": "_tmp/" } }] } }
}
```

Exclude a suffix:

```json
{
  "detail": { "object": { "key": [{ "anything-but": { "suffix": ".tmp" } }] } }
}
```

Exclude a wildcard:

```json
{
  "detail": { "object": { "key": [{ "anything-but": { "wildcard": "*/_SUCCESS*" } }] } }
}
```

Exclude case-insensitively:

```json
{
  "detail": { "object": { "key": [{ "anything-but": { "equals-ignore-case": "README.MD" } }] } }
}
```

> **Loop protection.** If your consumer writes results back into the *same* bucket, `anything-but` on the output prefix is what stops the infinite trigger loop. Better still, write to a different bucket.

### E.12 Numeric matching on object size

Only objects larger than 100 MB:

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": { "object": { "size": [{ "numeric": [">", 104857600] }] } }
}
```

A size band (between 1 KB and 5 GB):

```json
{
  "detail": { "object": { "size": [{ "numeric": [">=", 1024, "<=", 5368709120] }] } }
}
```

Zero-byte objects (the classic "folder placeholder" / failed-upload detector):

```json
{
  "detail": { "object": { "size": [{ "numeric": ["=", 0] }] } }
}
```

Operators: `=`, `!=`, `<`, `<=`, `>`, `>=`. Two operators max per matcher (to express a range).

### E.13 Field presence — `exists`

Only events that carry a size (excludes Tags/ACL events which have none):

```json
{
  "source": ["aws.s3"],
  "detail": { "object": { "size": [{ "exists": true }] } }
}
```

Only objects with **no** version id (bucket versioning off, or a non-versioned write):

```json
{
  "detail": { "object": { "version-id": [{ "exists": false }] } }
}
```

> `exists` works on **leaf** nodes only. `{"object": [{"exists": true}]}` on the parent object does not do what you expect.

### E.14 Filter by how the object was created — `reason`

Multipart uploads only (big-file pipelines):

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": { "reason": ["CompleteMultipartUpload"] }
}
```

Simple PUTs only, ignoring copies:

```json
{
  "detail-type": ["Object Created"],
  "detail": { "reason": ["PutObject"] }
}
```

Valid `reason` values for **Object Created**: `PutObject`, `POST Object`, `CopyObject`, `CompleteMultipartUpload`.
Valid `reason` values for **Object Deleted**: `DeleteObject`, `Lifecycle Expiration`.

> **This is the replacement for the old granular event types.** `s3:ObjectCreated:Put` becomes `detail-type = Object Created` **plus** `detail.reason = PutObject`. If you migrate a `:Put`-only notification to a bare `Object Created` rule, you will start receiving copies and multipart completions you never used to get.

### E.15 Deletes — distinguishing delete markers from real deletions

A delete marker on a versioned bucket (object still recoverable):

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Deleted"],
  "detail": { "deletion-type": ["Delete Marker Created"] }
}
```

A permanent deletion (data is gone — good candidate for an alert):

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Deleted"],
  "detail": { "deletion-type": ["Permanently Deleted"] }
}
```

Lifecycle-driven deletion versus a human/API deletion:

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Deleted"],
  "detail": { "reason": ["Lifecycle Expiration"] }
}
```

### E.16 Who did it — `requester` and `source-ip-address`

Exclude AWS internal actions (lifecycle, replication) and keep only real principals:

```json
{
  "source": ["aws.s3"],
  "detail": { "requester": [{ "anything-but": ["s3.amazonaws.com"] }] }
}
```

Alert on writes from outside your corporate CIDR:

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "source-ip-address": [{ "anything-but": { "cidr": "10.0.0.0/8" } }]
  }
}
```

Match a specific CIDR:

```json
{
  "detail": { "source-ip-address": [{ "cidr": "203.0.113.0/24" }] }
}
```

### E.17 Storage class / tiering transitions

An object moved to Glacier by lifecycle:

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Storage Class Changed"],
  "detail": { "destination-storage-class": ["GLACIER"] }
}
```

Intelligent-Tiering moved something to Archive Access:

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Access Tier Changed"],
  "detail": { "destination-access-tier": ["ARCHIVE_ACCESS"] }
}
```

### E.18 Restore lifecycle (Glacier workflows)

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Restore Initiated", "Object Restore Completed", "Object Restore Expired"],
  "detail": { "bucket": { "name": ["my-archive-bucket"] } }
}
```

Only completed restores, to kick off downstream processing:

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Restore Completed"]
}
```

### E.19 Tagging and ACL changes (compliance)

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Tags Added", "Object Tags Deleted", "Object ACL Updated"]
}
```

### E.20 Everything **except** creates and deletes (the "lifecycle noise" rule)

```json
{
  "source": ["aws.s3"],
  "detail-type": [{ "anything-but": ["Object Created", "Object Deleted"] }]
}
```

### E.21 Region / account scoping in a multi-account org bus

```json
{
  "source": ["aws.s3"],
  "account": ["111122223333", "444455556666"],
  "region": ["ap-south-1"],
  "detail-type": ["Object Created"]
}
```

### E.22 Match by bucket ARN via `resources`

```json
{
  "source": ["aws.s3"],
  "resources": ["arn:aws:s3:::my-bucket"]
}
```

### E.23 Exclude replayed events

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": { "replay-name": [{ "exists": false }] }
}
```

### E.24 A realistic production pattern combining several matchers

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": { "name": [{ "prefix": "prod-landing-" }] },
    "object": {
      "key": [{ "wildcard": "tenant-*/inbound/*.parquet" }],
      "size": [{ "numeric": [">", 0, "<", 5368709120] }]
    },
    "reason": ["PutObject", "CompleteMultipartUpload"],
    "requester": [{ "anything-but": ["s3.amazonaws.com"] }]
  }
}
```

Reads as: *a non-empty, sub-5GB parquet file written by a real principal into any tenant's inbound folder in any prod landing bucket*. Expressing this with S3 Event Notifications is impossible — not just inconvenient.

### E.25 Matcher support matrix

| Matcher | Syntax | Works on |
|---------|--------|----------|
| Exact | `["value"]` | Any field |
| Prefix | `[{"prefix":"raw/"}]` | Strings |
| Prefix, case-insensitive | `[{"prefix":{"equals-ignore-case":"RAW/"}}]` | Strings |
| Suffix | `[{"suffix":".csv"}]` | Strings |
| Suffix, case-insensitive | `[{"suffix":{"equals-ignore-case":".CSV"}}]` | Strings |
| Equals, case-insensitive | `[{"equals-ignore-case":"File.TXT"}]` | Strings |
| Wildcard | `[{"wildcard":"a/*/b*.csv"}]` | Strings, max 5 `*` |
| Anything-but (values) | `[{"anything-but":["x","y"]}]` | Strings, numbers |
| Anything-but (prefix/suffix/wildcard/ignore-case) | `[{"anything-but":{"prefix":"tmp/"}}]` | Strings |
| Numeric | `[{"numeric":[">",0,"<",100]}]` | Numbers, max 2 operators |
| Exists | `[{"exists":true}]` | Leaf nodes only |
| CIDR | `[{"cidr":"10.0.0.0/8"}]` | IP strings |
| Null | `[null]` | JSON null |
| Empty string | `[""]` | Strings |
| OR across fields | `"$or": [ {...}, {...} ]` | Whole sub-patterns |

---

## 12. Part F — Full event JSON samples for every S3 detail-type

S3 sends **all** of these to the bus once EventBridge is enabled. You cannot select a subset at the bucket; you filter with rules.

### F.0 The complete event type list

| `detail-type` | Fired when | Old S3 event type it replaces |
|---------------|-----------|-------------------------------|
| `Object Created` | Object written | `s3:ObjectCreated:Put`, `:Post`, `:Copy`, `:CompleteMultipartUpload` |
| `Object Deleted` | Object or version deleted, or lifecycle-expired | `s3:ObjectRemoved:Delete`, `:DeleteMarkerCreated`, `s3:LifecycleExpiration:*` |
| `Object Restore Initiated` | Restore requested from Glacier / IT archive tiers | `s3:ObjectRestore:Post` |
| `Object Restore Completed` | Restored copy is available | `s3:ObjectRestore:Completed` |
| `Object Restore Expired` | Temporary restored copy deleted | `s3:ObjectRestore:Delete` |
| `Object Storage Class Changed` | Lifecycle transition | `s3:LifecycleTransition` |
| `Object Access Tier Changed` | Intelligent-Tiering tier move | `s3:IntelligentTiering` |
| `Object Tags Added` | `PutObjectTagging` | `s3:ObjectTagging:Put` |
| `Object Tags Deleted` | `DeleteObjectTagging` | `s3:ObjectTagging:Delete` |
| `Object ACL Updated` | `PutObjectAcl` | `s3:ObjectAcl:Put` |
| `Async Copy Completion` | Asynchronous copy finishes | *(no direct equivalent)* |

> Not emitted by this integration: reads (`GetObject`), `HeadObject`, bucket-level config changes, `s3:ReducedRedundancyLostObject`. For reads and control-plane calls, use **CloudTrail → EventBridge** (`detail-type: "AWS API Call via CloudTrail"`).

### F.1 Object Created (simple PUT)

```json
{
  "version": "0",
  "id": "17793124-05d4-b198-2fde-7ededc63b103",
  "detail-type": "Object Created",
  "source": "aws.s3",
  "account": "111122223333",
  "time": "2026-09-17T09:12:41Z",
  "region": "ap-south-1",
  "resources": ["arn:aws:s3:::my-bucket"],
  "detail": {
    "version": "0",
    "bucket": { "name": "my-bucket" },
    "object": {
      "key": "raw/report.csv",
      "size": 1048576,
      "etag": "b1946ac92492d2347c6235b4d2611184",
      "version-id": "IYV3p45BT0ac8hjHg1houSdS1a.Mro8e",
      "sequencer": "617f08299329d189"
    },
    "request-id": "N4N7GDK58NMKJ12R",
    "requester": "111122223333",
    "source-ip-address": "203.0.113.42",
    "reason": "PutObject"
  }
}
```

### F.2 Object Created (multipart upload)

```json
{
  "detail-type": "Object Created",
  "source": "aws.s3",
  "detail": {
    "bucket": { "name": "my-bucket" },
    "object": {
      "key": "bigdata/dump-2026-09-17.tar.gz",
      "size": 10737418240,
      "etag": "d41d8cd98f00b204e9800998ecf8427e-64",
      "sequencer": "0062E99A88DC407460"
    },
    "request-id": "PQ8R2T4V6X8Z0B2D",
    "requester": "111122223333",
    "reason": "CompleteMultipartUpload"
  }
}
```

> The `-64` suffix on the etag is the part count. An etag with a dash is **not** an MD5 of the object — do not use it for integrity checks on multipart objects.

### F.3 Object Created (server-side copy)

```json
{
  "detail-type": "Object Created",
  "source": "aws.s3",
  "detail": {
    "bucket": { "name": "my-bucket" },
    "object": { "key": "archive/report.csv", "size": 1048576, "etag": "b1946ac9..." },
    "reason": "CopyObject"
  }
}
```

### F.4 Object Deleted (delete marker on a versioned bucket)

```json
{
  "detail-type": "Object Deleted",
  "source": "aws.s3",
  "detail": {
    "version": "0",
    "bucket": { "name": "my-bucket" },
    "object": {
      "key": "raw/report.csv",
      "etag": "d41d8cd98f00b204e9800998ecf8427e",
      "version-id": "IYV3p45BT0ac8hjHg1houSdS1a.Mro8e",
      "sequencer": "617f0831b8b9d3a7"
    },
    "request-id": "0B4A7C9E1F3A5C7E",
    "requester": "111122223333",
    "source-ip-address": "203.0.113.42",
    "reason": "DeleteObject",
    "deletion-type": "Delete Marker Created"
  }
}
```

> **Note the missing `size`.** Delete events do not carry it. Any consumer doing `detail["object"]["size"]` without `.get()` will crash on the first delete.

### F.5 Object Deleted (permanent, lifecycle expiration)

```json
{
  "detail-type": "Object Deleted",
  "source": "aws.s3",
  "detail": {
    "bucket": { "name": "my-bucket" },
    "object": { "key": "tmp/scratch.bin", "etag": "abc123", "sequencer": "617f0832..." },
    "requester": "s3.amazonaws.com",
    "reason": "Lifecycle Expiration",
    "deletion-type": "Permanently Deleted"
  }
}
```

> `requester` is `s3.amazonaws.com`, not an account id, and there is **no** `source-ip-address` — the event was not triggered by a request. Filter on this to separate automated from human activity.

### F.6 Object Restore Initiated / Completed / Expired

```json
{
  "detail-type": "Object Restore Completed",
  "source": "aws.s3",
  "detail": {
    "bucket": { "name": "my-archive-bucket" },
    "object": { "key": "cold/2019/ledger.zip", "size": 52428800, "etag": "abc123", "version-id": "..." },
    "request-id": "189F19CB7FB1B6A4",
    "requester": "s3.amazonaws.com",
    "restore-expiry-time": "2026-09-24T09:12:41Z",
    "source-storage-class": "GLACIER"
  }
}
```

`Object Restore Initiated` carries `restore-expiry-time` absent and `requester` set to the caller. `Object Restore Expired` carries neither size nor expiry.

### F.7 Object Storage Class Changed (lifecycle transition)

```json
{
  "detail-type": "Object Storage Class Changed",
  "source": "aws.s3",
  "detail": {
    "bucket": { "name": "my-bucket" },
    "object": { "key": "logs/2026/01/app.log", "size": 4096, "etag": "abc123", "version-id": "..." },
    "requester": "s3.amazonaws.com",
    "destination-storage-class": "GLACIER_IR"
  }
}
```

### F.8 Object Access Tier Changed (Intelligent-Tiering)

```json
{
  "detail-type": "Object Access Tier Changed",
  "source": "aws.s3",
  "detail": {
    "bucket": { "name": "my-bucket" },
    "object": { "key": "media/clip.mov", "size": 734003200, "etag": "abc123" },
    "requester": "s3.amazonaws.com",
    "destination-access-tier": "ARCHIVE_ACCESS"
  }
}
```

### F.9 Object Tags Added / Deleted

```json
{
  "detail-type": "Object Tags Added",
  "source": "aws.s3",
  "detail": {
    "bucket": { "name": "my-bucket" },
    "object": { "key": "raw/report.csv", "etag": "abc123", "version-id": "..." },
    "request-id": "AB12CD34EF56",
    "requester": "111122223333",
    "source-ip-address": "203.0.113.42"
  }
}
```

### F.10 Object ACL Updated

```json
{
  "detail-type": "Object ACL Updated",
  "source": "aws.s3",
  "detail": {
    "bucket": { "name": "my-bucket" },
    "object": { "key": "public/logo.png", "etag": "abc123", "version-id": "..." },
    "requester": "111122223333",
    "source-ip-address": "203.0.113.42"
  }
}
```

A high-value security rule: alert whenever an ACL is changed on a bucket that is supposed to be fully private.

### F.11 Field availability matrix

| Field | Created | Deleted | Restore Init/Compl/Exp | Storage Class Changed | Access Tier Changed | Tags Added/Deleted | ACL Updated |
|-------|:-------:|:-------:|:----------------------:|:---------------------:|:-------------------:|:------------------:|:-----------:|
| `object.key` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `object.size` | ✅ | ❌ | ✅ / ✅ / ❌ | ✅ | ✅ | ❌ | ❌ |
| `object.etag` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `object.version-id` | versioned only | versioned only | versioned only | versioned only | versioned only | versioned only | versioned only |
| `object.sequencer` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `reason` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `deletion-type` | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `source-ip-address` | request-driven only | request-driven only | request-driven only | ❌ | ❌ | ✅ | ✅ |
| `destination-storage-class` | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| `destination-access-tier` | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| `restore-expiry-time` | ❌ | ❌ | Completed only | ❌ | ❌ | ❌ | ❌ |

**Write every consumer defensively.** Use `.get()` / optional chaining for `size`, `version-id`, `reason`, `source-ip-address`.

---

## 13. Part G — CloudFormation

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: S3 to EventBridge - bucket flag, rule, Lambda target, DLQ

Parameters:
  BucketName:
    Type: String
    Description: Name of the bucket to create with EventBridge enabled

Resources:

  LabBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref BucketName
      VersioningConfiguration:
        Status: Enabled
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      NotificationConfiguration:
        EventBridgeConfiguration:
          EventBridgeEnabled: true      # <-- the whole migration, in one property

  EventsDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: s3-events-dlq
      MessageRetentionPeriod: 1209600

  EventsDLQPolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref EventsDLQ
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Sid: AllowEventBridgeDLQ
            Effect: Allow
            Principal:
              Service: events.amazonaws.com
            Action: sqs:SendMessage
            Resource: !GetAtt EventsDLQ.Arn
            Condition:
              ArnEquals:
                aws:SourceArn: !GetAtt RawCsvRule.Arn

  RawCsvRule:
    Type: AWS::Events::Rule
    Properties:
      Name: s3-raw-csv-created
      Description: raw/*.csv creations
      State: ENABLED
      EventBusName: default
      EventPattern:
        source:
          - aws.s3
        detail-type:
          - Object Created
        detail:
          bucket:
            name:
              - !Ref BucketName
          object:
            key:
              - wildcard: "raw/*.csv"
          reason:
            - PutObject
            - CompleteMultipartUpload
      Targets:
        - Id: csv-processor
          Arn: !GetAtt CsvProcessor.Arn
          RetryPolicy:
            MaximumEventAgeInSeconds: 3600
            MaximumRetryAttempts: 5
          DeadLetterConfig:
            Arn: !GetAtt EventsDLQ.Arn

  CsvProcessorPermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref CsvProcessor
      Action: lambda:InvokeFunction
      Principal: events.amazonaws.com
      SourceArn: !GetAtt RawCsvRule.Arn

  CsvProcessorRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

  CsvProcessor:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: process-raw-csv
      Runtime: python3.12
      Handler: index.lambda_handler
      Role: !GetAtt CsvProcessorRole.Arn
      Timeout: 30
      Code:
        ZipFile: |
          import json
          def lambda_handler(event, context):
              d = event["detail"]
              print(json.dumps({
                  "bucket": d["bucket"]["name"],
                  "key": d["object"]["key"],
                  "size": d["object"].get("size"),
                  "reason": d.get("reason"),
              }))
              return {"ok": True}

Outputs:
  BucketArn:
    Value: !GetAtt LabBucket.Arn
  RuleArn:
    Value: !GetAtt RawCsvRule.Arn
```

> **The same authoritative-config caveat applies here as with any infrastructure-as-code tool.** `AWS::S3::Bucket` manages the whole `NotificationConfiguration`. You cannot use CloudFormation to add *only* the EventBridge flag to a bucket whose other notifications were made elsewhere — it will remove them. For an existing bucket, enable the flag from the console instead (B.2), or import the bucket into the stack with every notification declared.

---

## 14. Part H — Migration playbook: dual-run, cutover, rollback

This is how you move 100 live notifications without an outage.

### H.1 The principle

Enabling EventBridge does **not** disable existing notifications. Both fire. That overlap is a feature: it gives you a dual-run window where you can validate the new path against the old one before removing anything.

```
   Phase 1            Phase 2               Phase 3            Phase 4
   Baseline           Dual-run              Cutover            Cleanup

  S3 ──► SQS/λ       S3 ──┬─► SQS/λ (live) S3 ──┬─► SQS/λ     S3 ──► EB ──► targets
                          └─► EB ──► shadow     └─► EB (live)
                               (log only)            ▲
                                                  swap here
```

### H.2 Step-by-step

| Step | Action | Verify before moving on |
|------|--------|-------------------------|
| 1 | Export and archive the notification config (A.1) | Backup file exists in version control |
| 2 | Build the routing matrix (A.4) | Every old config has an owner and a planned rule name |
| 3 | Enable EventBridge on the bucket (B.2) — merge into the existing config, never overwrite it | `EventBridgeConfiguration` present **and** old configs count unchanged |
| 4 | Create an **archive** on the bus (D.4) | Archive state `ENABLED`; events accumulating |
| 5 | Create the catch-all debug rule → CloudWatch Logs | Events visible within ~5 minutes |
| 6 | Build new rules pointing at **shadow** targets (a copy of each consumer that only logs) | Shadow output matches production output 1:1 for a full business cycle |
| 7 | Compare counts: `MatchedEvents` per rule vs old consumer's processed count | Deltas explained (see H.3) |
| 8 | Repoint rules from shadow to the real targets, one consumer at a time | Consumer healthy, DLQ empty |
| 9 | Delete the corresponding **old** notification config entry (merge-delete, never wholesale PUT) | Slot count drops by one; consumer still receiving |
| 10 | Repeat 8–9 per consumer until the old config is empty | Slot count → 0 |
| 11 | Delete the catch-all debug rule and its log group | CloudWatch Logs cost stops |
| 12 | Keep the archive for one retention period, then drop it | — |

### H.3 Expected count deltas during comparison (these are not bugs)

| Symptom | Cause |
|---------|-------|
| EventBridge sees **more** creates | Your old config was `s3:ObjectCreated:Put` only; EventBridge `Object Created` also includes Copy / POST / CompleteMultipartUpload. Add `detail.reason` to the pattern |
| EventBridge sees **more** deletes | Old config was `ObjectRemoved:Delete`; EventBridge also emits lifecycle expirations. Filter on `reason` / `deletion-type` |
| EventBridge sees events the old path never did | Tags, ACL, storage-class and tiering events are always published now. Your rules must be specific |
| Keys with spaces differ | Old path was URL-encoded (`a+b.csv`), EventBridge is not (`a b.csv`). Remove `unquote_plus` from consumers |
| Consumer processes each object twice | Dual-run is working exactly as designed. Make consumers idempotent before step 6 |

### H.4 Removing one old notification safely

1. Console → **S3** → your bucket → **Properties** → **Event notifications**
2. Find the old notification by name (from your A.4 migration matrix)
3. Click its name → **Delete** → confirm

Because the console adds and removes notifications one at a time, this only ever touches the single entry you selected — the other 99 stay exactly as they were.

### H.5 Rollback at any point

| What broke | Rollback |
|------------|----------|
| A single new rule misbehaves | `aws events disable-rule --name <rule>` — instant, reversible, no delete |
| A target is flooding a consumer | `aws events remove-targets --rule <rule> --ids <id>` |
| The whole EventBridge path is wrong | Remove `EventBridgeConfiguration` from the bucket's notification config using the same merge-delete method as H.4 — old notifications keep working untouched |
| You deleted old notifications too early | Restore from the A.1 backup file with a full PUT |
| You lost events during a gap | Replay from the archive (D.4) |

The archive in step 4 is the reason step 9 is safe. Do not skip it.

---

## 15. Part I — Edge cases and gotchas

| # | Gotcha | Detail | What to do |
|---|--------|--------|-----------|
| 1 | **The underlying API is a full replace, even though the console isn't** | Scripting this yourself (SDK, IaC) with a partial document wipes every existing notification, silently and instantly | Use the console (adds/removes one at a time), or if scripting, always read-merge-write |
| 2 | **~5 minute propagation** | Events may not flow for about five minutes after enabling | Wait before declaring failure |
| 3 | **No test event** | SQS/SNS get an `s3:TestEvent` on setup; EventBridge gets nothing | Verify by uploading a real object |
| 4 | **Object keys are not URL-encoded** | Old format encoded them (space → `+`) | Delete `unquote_plus` from every consumer; test with a key containing a space, `+`, `%` and `#` |
| 5 | **One event per invocation** | No `Records[]` array, ever | Remove all `for record in event['Records']` loops |
| 6 | **Same-field array is OR, not AND** | `key: [{"prefix":"a/"},{"suffix":".csv"}]` matches `a/x.txt` | Use `{"wildcard":"a/*.csv"}` |
| 7 | **Granular create types are gone** | `ObjectCreated:Put` → `Object Created` + `reason` | Add `detail.reason` or you will process copies you never wanted |
| 8 | **Lifecycle deletes now reach you** | They were a separate old event type | Filter on `reason` / `deletion-type` |
| 9 | **`size` is missing on delete/tag/ACL events** | `KeyError` on the first delete | Use `.get("size")` everywhere |
| 10 | **Default bus only** | S3 cannot publish to a custom bus | Forward with a rule if you need one |
| 11 | **Same region only** | S3 publishes to the bus in the bucket's region | Cross-region needs bus-to-bus forwarding + a role |
| 12 | **300 rules per bus** | Adjustable, but not infinite | Consolidate: one rule with 5 targets, or fan out via SNS, or route in a dispatcher |
| 13 | **5 targets per rule is not adjustable** | Hard cap | Duplicate the rule, or fan out with SNS |
| 14 | **Infinite loops** | Consumer writes back into the same bucket, retriggering itself | Different bucket, or `anything-but` on the output prefix |
| 15 | **No ordering guarantee** | Even with FIFO targets | Compare `sequencer` lexicographically (pad to equal length); larger = later |
| 16 | **At-least-once delivery** | Duplicates are normal | Idempotency key: `event.id`, or `bucket+key+sequencer` |
| 17 | **Events dropped with no DLQ** | Retries exhaust and the event vanishes | Attach a DLQ to every production target |
| 18 | **CloudWatch Logs target needs `:*`** | ARN must end with `:*` | `...:log-group:/aws/events/x:*` |
| 19 | **CloudWatch Logs needs a resource policy** | The console adds it for you when you pick a log group target; scripting it yourself does not | Console handles this automatically (B.3) |
| 20 | **KMS-encrypted SQS/SNS targets fail silently** | `events.amazonaws.com` lacks key access | Add `kms:GenerateDataKey*` + `kms:Decrypt` to the key policy |
| 21 | **FIFO targets need `MessageGroupId`** | Otherwise the target call fails | Set it on the target's `SqsParameters` |
| 22 | **Infrastructure-as-code tools own the whole notification config** | Same wipe risk as #1 if you manage the bucket with IaC elsewhere | Import the bucket first, or declare every existing notification in the same resource |
| 23 | **Directory buckets (S3 Express One Zone)** | `PutBucketNotificationConfiguration` is unsupported there | Not available; use a different pattern |
| 24 | **Replayed events re-fire live rules** | Consumers see historical events as new | Idempotency, or exclude with `replay-name: [{"exists": false}]` |
| 25 | **`exists` only works on leaf nodes** | Parent-object checks silently never match | Test with `aws events test-event-pattern` |
| 26 | **Multipart etag is not an MD5** | Etag ends `-<partcount>` | Do not use it for integrity checks |
| 27 | **Lifecycle/replication events have `requester: s3.amazonaws.com`** | And no `source-ip-address` | Filter with `anything-but` to isolate human activity |
| 28 | **Catch-all → CloudWatch Logs is expensive** | Ingestion charges on every event | Debug only; delete it |
| 29 | **Event size cap 256 KB** | S3 events are far smaller, so not a practical issue here | Relevant only if you forward enriched events |
| 30 | **Rules are evaluated independently** | Ten matching rules = ten deliveries | Intentional — but be aware when estimating downstream load |
| 31 | **Disabled rule still costs nothing but silently drops** | Easy to forget after a rollback | Alarm on `MatchedEvents = 0` for rules that should be busy |
| 32 | **Testing a pattern needs a complete sample event** | Missing `version`/`id`/`account` etc. makes the tester reject it | Use the **Sample event** field when creating a rule, or copy a full event skeleton from Part F |

---

## 16. Part J — Troubleshooting table

| Symptom | Likely cause | How to confirm | Fix |
|---------|--------------|----------------|-----|
| No events at all, anywhere | EventBridge not enabled on the bucket | Properties → Event notifications → EventBridge panel shows **Off** | Turn it **On** (B.2) |
| No events, flag is set | Still within the ~5 min propagation window | Wait and retry | Patience |
| No events, flag set, 10+ min | Rule is in a different region than the bucket | Compare the bucket's Region (A.5) with your rule's event bus region | Recreate the rule in the bucket's region |
| `MatchedEvents = 0` | Event pattern does not match | Paste a real event from the catch-all log into the rule's Sample event / pattern tester | Fix the pattern; check array-vs-scalar and OR-vs-AND |
| Pattern looks right but never matches | Value written as a scalar instead of an array | `"name": "x"` instead of `"name": ["x"]` | Wrap in an array |
| Prefix+suffix rule never matches | Two matchers on one field are OR'ed | Test with a key that matches only the prefix | Use `wildcard` |
| `MatchedEvents > 0`, `Invocations = 0` | Target permission missing | Rule's Monitoring tab: check `FailedInvocations`; check the DLQ `ERROR_CODE` | Add the resource policy / role (Part C) |
| `FailedInvocations` rising, DLQ empty | No DLQ configured — events are being lost | Rule → Targets → check for a Dead-letter queue | Attach a DLQ immediately |
| DLQ messages with `ERROR_CODE=NO_PERMISSIONS` | Role or resource policy wrong | Read the `ERROR_MESSAGE` message attribute in the DLQ | Fix the policy; re-drive the DLQ |
| Log group exists but stays empty | Missing CloudWatch Logs resource policy, or ARN missing `:*` | Confirm the target ARN ends `:*` | Recreate the target with the correct ARN (B.3) |
| Lambda not invoked, no error anywhere | Function's resource-based policy has no EventBridge statement | Lambda console → function → **Configuration** → **Permissions** | Recreate the rule/target from the console so it adds the permission (C.1) |
| SQS target fails silently, queue is KMS-encrypted | Key policy blocks `events.amazonaws.com` | CloudTrail shows `kms:GenerateDataKey` AccessDenied | Update the key policy |
| FIFO queue target fails | `MessageGroupId` not set | Rule → Targets → the FIFO queue's configuration | Set a Message group ID on the target (C.3) |
| Consumer crashes with `KeyError: 'Records'` | Code still expects the old S3 format | Inspect the payload | Rewrite to `event["detail"]` (C.1) |
| Consumer crashes with `KeyError: 'size'` | Delete / tag / ACL event reached a create-only handler | Check `detail-type` in the log | Tighten the pattern **and** use `.get()` |
| Keys with spaces resolve to the wrong object | Consumer still calls `unquote_plus` | Upload `my file.txt` and compare | Remove the unquote |
| Everything processed twice | Old notification and the new rule both live | Expected during dual-run | Finish H.2 step 9 |
| Objects processed twice with no dual-run | At-least-once delivery, or two overlapping rules | Compare `event.id` values | Idempotency; audit overlapping patterns |
| Can't create another rule | 300 rules on the bus | EventBridge → Rules — count the list | Consolidate, or request a Service Quotas increase |
| Rule won't save, "too many targets" | 6th target on one rule | Count the targets listed on the rule | Split into another rule |
| Infinite invocations, cost spike | Consumer writes back into the source bucket | Look at the key pattern in the logs | Output to another bucket, or `anything-but` |
| Events arrive but hours late | Target throttling / retry backoff | Rule's Monitoring tab: `ThrottledRules`, `IngestionToInvocationStartLatency` | Raise target concurrency; request a quota increase |
| Cannot add EventBridge via infrastructure-as-code without destroying notifications | The bucket's notification resource is authoritative in most IaC tools | Plan/diff output shows removals | Import first, or declare every existing notification (see Part F — CloudFormation) |
| Old notifications reappeared | Someone re-applied an old infrastructure-as-code template that manages the bucket | CloudTrail shows `PutBucketNotificationConfiguration` | Remove or update the stale template |

---

## 17. Part K — Monitoring and alarms

### K.1 Metrics that matter (namespace `AWS/Events`)

| Metric | Dimension | Read it as |
|--------|-----------|-----------|
| `MatchedEvents` | RuleName | Pattern is matching. **0 = pattern problem** |
| `TriggeredRules` | RuleName | Legacy equivalent of MatchedEvents |
| `Invocations` | RuleName | Target was called |
| `FailedInvocations` | RuleName | Target call failed permanently. **Alarm on this** |
| `ThrottledRules` | RuleName | Hit an invocation quota |
| `DeadLetterInvocations` | RuleName | Events landed in the DLQ. **Alarm on this** |
| `InvocationsSentToDlq` | RuleName | Same family |
| `InvocationsFailedToBeSentToDlq` | RuleName | **Event lost entirely — page someone** |
| `IngestionToInvocationStartLatency` | EventBusName | End-to-end lag on the bus |

Remember the diagnostic split: **MatchedEvents ⇒ pattern**, **Invocations/FailedInvocations ⇒ permissions**.

### K.2 Alarms to create

For each alarm: **CloudWatch** console → **Alarms** → **All alarms** → **Create alarm** → **Select metric** → **AWS/Events** → **By Rule Name** → pick the rule and metric below.

| Alarm | Metric | Rule | Statistic / Period | Threshold |
|-------|--------|------|---------------------|-----------|
| `eb-s3-failed-invocations` | `FailedInvocations` | `s3-raw-csv-to-lambda` | Sum / 5 min | ≥ 1 for 1 period |
| `eb-s3-dlq-invocations` | `DeadLetterInvocations` | `s3-raw-csv-to-lambda` | Sum / 5 min | ≥ 1 for 1 period |
| `eb-s3-no-matches` | `MatchedEvents` | `s3-raw-csv-to-lambda` | Sum / 1 hour | < 1 for 2 periods |

For each, set **Missing data treatment**: "not breaching" for the first two, "breaching" for the third (so a total gap in data still fires the "no matches" alarm). Attach an SNS topic on the **Notification** step so the alarm actually reaches someone.

The third one — "this rule has matched nothing for two hours" — is the alarm that catches a silently broken migration. Most teams only build the first two and never notice a rule that stopped matching after a pattern edit.

### K.3 Track your rule count against the quota

**EventBridge** console → **Rules** (with **Event bus = default**) shows every rule; the count of the filtered list is your current usage. For the quota itself, go to **Service Quotas** console → **AWS services** → search **Amazon EventBridge** → **Rules per event bus** — this page shows your current limit and lets you request an increase (see Part L).

### K.4 Auditing config changes

Create a rule that watches for anyone modifying the bucket notification configuration (requires CloudTrail management events, which are on by default):

```json
{
  "source": ["aws.s3"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["s3.amazonaws.com"],
    "eventName": ["PutBucketNotificationConfiguration"]
  }
}
```

After the effort of this migration, knowing immediately when someone PUTs over your configuration is worth the one rule it costs.

---

## 18. Part L — Security and IAM

### L.1 Minimum permissions to run this lab

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3Config",
      "Effect": "Allow",
      "Action": [
        "s3:CreateBucket",
        "s3:PutBucketNotification",
        "s3:GetBucketNotification",
        "s3:GetBucketLocation",
        "s3:PutBucketVersioning",
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:DeleteObjectVersion",
        "s3:ListBucket",
        "s3:ListBucketVersions"
      ],
      "Resource": [
        "arn:aws:s3:::ebridge-lab-*",
        "arn:aws:s3:::ebridge-lab-*/*"
      ]
    },
    {
      "Sid": "EventBridge",
      "Effect": "Allow",
      "Action": [
        "events:PutRule",
        "events:PutTargets",
        "events:DeleteRule",
        "events:RemoveTargets",
        "events:DescribeRule",
        "events:ListRules",
        "events:ListTargetsByRule",
        "events:EnableRule",
        "events:DisableRule",
        "events:TestEventPattern",
        "events:CreateArchive",
        "events:DescribeArchive",
        "events:DeleteArchive",
        "events:StartReplay",
        "events:DescribeReplay",
        "events:TagResource"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Targets",
      "Effect": "Allow",
      "Action": [
        "lambda:AddPermission",
        "lambda:RemovePermission",
        "lambda:GetPolicy",
        "sqs:CreateQueue",
        "sqs:SetQueueAttributes",
        "sqs:GetQueueAttributes",
        "sqs:DeleteQueue",
        "sns:CreateTopic",
        "sns:SetTopicAttributes",
        "sns:GetTopicAttributes",
        "sns:DeleteTopic",
        "logs:CreateLogGroup",
        "logs:DeleteLogGroup",
        "logs:PutResourcePolicy",
        "logs:DescribeResourcePolicies",
        "logs:PutRetentionPolicy",
        "iam:PassRole"
      ],
      "Resource": "*"
    }
  ]
}
```

> Note `s3:PutBucketNotification` — the IAM action name has no "Configuration" suffix even though the API call does. Getting this wrong is a common `AccessDenied`.

> Scope `iam:PassRole` in production with `Condition: {"StringEquals": {"iam:PassedToService": "events.amazonaws.com"}}`.

### L.2 Security principles for this design

1. **Always condition resource policies on `aws:SourceArn`.** Without it, any rule in any AWS account can push to your queue or topic.
2. **Add `aws:SourceAccount`** as well for roles assumed by `events.amazonaws.com`.
3. **Least privilege per target role** — one role per rule/target pair, not one shared `EventBridgeInvokeAll` role.
4. **Encrypt the DLQ.** It contains full event payloads including key names, which are often sensitive.
5. **Object keys can leak data.** `customers/aadhaar-1234-5678.pdf` ends up in CloudWatch Logs, DLQs and SNS emails. Treat key names as sensitive in every downstream store.
6. **The catch-all rule is a data exfiltration surface.** Anyone with `logs:GetLogEvents` on `/aws/events/*` sees every object name in the bucket. Delete it after debugging.
7. **Cross-account forwarding** requires a bus resource policy — review it like a bucket policy, with explicit account principals, never `"Principal": "*"`.
8. **Enable CloudTrail management events** so `PutBucketNotificationConfiguration` and `PutRule` are auditable.
9. **EventBridge default bus encryption** — you can attach a customer managed KMS key to the bus if your compliance posture requires it.

---

## 19. Part M — Billing variables

| Component | Charge | Notes for this design |
|-----------|--------|----------------------|
| S3 Event Notifications | **Free** | Never the cost driver |
| S3 → EventBridge publishing | **Free** | Events published by AWS services to the default bus are not billed |
| EventBridge rule evaluation | **Free** | Rules and matching cost nothing |
| Custom events (`PutEvents`) | ~$1.00 per million | Not used here unless you forward |
| **Cross-account / cross-region forwarding** | Billed as custom events | The one EventBridge line item that can surprise you |
| API Destinations | ~$0.20 per million invocations | Only if you use them |
| Archive storage | ~$0.10 per GB-month | Retention × event volume |
| Replay | ~$0.10 per GB processed | One-off during migration |
| Schema discovery | Per event ingested | Leave it off |
| **Lambda targets** | Requests + GB-seconds | Unchanged from before — same invocations |
| **SQS targets** | Per request (first 1M/month free) | Unchanged |
| **SNS targets** | Per publish + per delivery | Unchanged |
| **CloudWatch Logs target** | **~$0.50–0.90 per GB ingested** + storage | ⚠️ The catch-all rule is the expensive one |
| Step Functions targets | Per state transition | Standard workflows can be pricey at high event rates |
| S3 requests | Unchanged | The migration does not add S3 requests |

### Cost model in one line

> The EventBridge plumbing is effectively free. **Your bill moves to the targets**, and it moves *up* if you now match events you previously filtered out at the bucket.

