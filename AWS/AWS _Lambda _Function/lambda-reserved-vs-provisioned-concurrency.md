# AWS Lambda: Reserved Concurrency vs Provisioned Concurrency
### Simple End-to-End Guide

---

## 1. The Basic Idea First

Every AWS account has a shared pool of **Lambda concurrency** per Region.

```
Default account concurrency limit = 1,000 concurrent executions
(shared by ALL Lambda functions in that Region)
```

Every time your function is invoked while a previous invocation is still running, Lambda spins up a **new execution environment** — that's 1 unit of concurrency. If 50 requests hit your function at the same time, that's 50 concurrent executions.

**The problem this causes:** all your functions fight over the same 1,000-slot pool. One noisy function can eat all the slots and starve everyone else.

This is why AWS gives you two controls: **Reserved Concurrency** and **Provisioned Concurrency**.

---

## 2. Reserved Concurrency — "Guaranteed Lane + Speed Limit"

### What it does
Reserved Concurrency does **two things at once**:
1. **Guarantees** a slice of the account pool exclusively for your function (no one else can use it).
2. **Caps** your function so it can never scale beyond that number.

### Simple analogy
Think of a highway with a **reserved lane**. That lane is yours — no other car can use it. But you also can't drive in the other lanes. So it's a guarantee AND a limit at the same time.

### Key facts

| Fact | Detail |
|---|---|
| Cost | **Free** — no extra charge |
| Effect on cold starts | **No help** — does not pre-warm anything |
| Effect on other functions | Reduces the shared pool available to everyone else |
| Set to 0 | Instantly stops the function from running (kill switch) |
| Who should use it | Functions that must never overwhelm a downstream system (e.g., RDS database with limited connections) |

### Min / Max values

| Setting | Value |
|---|---|
| Minimum | 0 |
| Default | Not set — function uses the shared account pool |
| Default account concurrency | 1,000 concurrent executions / Region |
| **Maximum you can reserve (default account quota)** | **900** |
| Unreserved concurrency AWS always keeps aside | 100 |

```
Total account concurrency = 1,000

Reserved concurrency
        ↓
   Maximum = 900

Remaining unreserved
        ↓
       100
```

> AWS always keeps **100 concurrency unreserved**, so you can never reserve the full 1,000 — even if no other function has reserved anything. If your account-level quota is increased above 1,000, the reservable max increases too, but 100 stays unreserved.

**Easy interview answer:** Reserved Concurrency minimum is 0. With the default Lambda account concurrency of 1,000 per Region, you can reserve up to 900 for a function, because Lambda always keeps 100 unreserved for functions without a reservation.

### Example
```
Account concurrency limit = 1,000
Reserved concurrency for "order-processor" = 100

Result:
- order-processor can run up to 100 times at once, never more
- Those 100 slots are locked for order-processor only
- Remaining 900 slots are shared by all other functions
```

### Special case — setting it to 0
```
Reserved Concurrency = 0
```
This effectively **disables** the function — it won't process any invocations until you remove or raise the setting. Useful as an emergency kill switch.

### How to set it (Console)
```
Lambda Console → Function → Configuration → Concurrency → Reserve concurrency → Enter number → Save
```

### How to set it (CLI)
```bash
aws lambda put-function-concurrency \
  --function-name order-processor \
  --reserved-concurrent-executions 100
```

### Remove it
```bash
aws lambda delete-function-concurrency \
  --function-name order-processor
```

---

## 3. Provisioned Concurrency — "Pre-Warmed Engines Ready to Go"

### What it does
Provisioned Concurrency **pre-initializes** a set number of execution environments so they are **always warm and ready**, eliminating cold starts for that many concurrent requests.

### Simple analogy
Like keeping a few taxis with engines running outside the airport, so passengers get in and go instantly — instead of calling a taxi and waiting for it to arrive (cold start).

### Key facts

| Fact | Detail |
|---|---|
| Cost | **Extra charge** — you pay for idle warm time + invocation time |
| Effect on cold starts | **Eliminates them** for the provisioned amount |
| Requires | A published **function version or alias** (not `$LATEST`) |
| Max you can set | Unreserved account concurrency **minus 100** |
| Relationship with Reserved | Cannot exceed the Reserved Concurrency value if both are set on the same function |

### Example
```
Account concurrency limit = 1,000
No other reservations exist

Max provisioned concurrency for one function = 1,000 - 100 = 900
(AWS always keeps 100 slots aside for functions with no reservation)
```

### Combining both (most common real setup)
```
Reserved concurrency  = 100   → hard ceiling, guaranteed slots
Provisioned concurrency = 50  → 50 of those slots are pre-warmed

Result:
- First 50 requests → instant (warm, no cold start)
- Requests 51-100    → cold start, but still allowed (within reserved ceiling)
- Request 101+       → throttled (reserved ceiling reached)
```

### How to set it (Console)
```
Lambda Console → Function → Versions → Publish new version 
→ Create alias pointing to that version
→ Configuration → Concurrency → Provisioned concurrency → Enter number → Save
```

### How to set it (CLI)
```bash
# 1. Publish a version
aws lambda publish-version --function-name order-processor

# 2. Create an alias pointing to that version
aws lambda create-alias \
  --function-name order-processor \
  --name prod \
  --function-version 1

# 3. Set provisioned concurrency on the alias
aws lambda put-provisioned-concurrency-config \
  --function-name order-processor \
  --qualifier prod \
  --provisioned-concurrent-executions 50
```

### Check status
```bash
aws lambda get-provisioned-concurrency-config \
  --function-name order-processor \
  --qualifier prod
```

Status values: `IN_PROGRESS` (allocating) → `READY` (available) → `FAILED` (check `StatusReason`)

### Remove it
```bash
aws lambda delete-provisioned-concurrency-config \
  --function-name order-processor \
  --qualifier prod
```

---

## 4. Is Concurrency Paid?

**No — concurrency itself is not charged separately.** Lambda bills you for what your function actually *does*, not for the slots it's allowed to use.

| Feature | Separate charge? |
|---|---|
| Concurrency (in general) | ❌ No |
| Reserved Concurrency | ❌ No |
| Provisioned Concurrency | ✅ Yes |
| Lambda execution / requests | ✅ Yes |

### Example
```
Reserved Concurrency = 100
        ↓
No separate "100 concurrency" charge
```
If you reserve 100 but the function never runs, **you pay nothing extra**. You only pay for actual invocations (requests + duration + memory).

Provisioned Concurrency is the exception — AWS charges you for keeping environments **pre-initialized and idle**, whether or not you use them.

```
Reserved Concurrency     → No separate capacity charge
Provisioned Concurrency  → Additional charge for keeping environments ready
```

---

## 5. Lambda Billing Variables (What Actually Costs Money)

| Variable | What it means |
|---|---|
| **Requests** | How many times your function is invoked |
| **Duration** | How long each invocation runs |
| **Memory** | Memory configured for the function |
| **Architecture** | x86 vs ARM/Graviton — different pricing |
| **Provisioned Concurrency** | Extra cost for keeping environments warm |
| **Ephemeral storage** | Extra `/tmp` storage beyond the included amount |
| **Other services** | API Gateway, S3, DynamoDB, CloudWatch, etc. bill separately |

### Simple formula
```
Lambda Cost
   =
Number of Requests
+
(Memory allocated × Execution Duration)

Then add, if applicable:
+ Provisioned Concurrency
+ Additional Ephemeral Storage
+ Other AWS services used
```

### Example
```
Requests       = 1,000,000
Memory         = 512 MB
Average time   = 2 seconds
Provisioned    = 0

Cost driven by:
1,000,000 requests
        +
512 MB × 2 sec × those executions

NOT by Reserved Concurrency.
```

> **Note:** Higher memory also gives your function proportionally more CPU — so a higher-memory config can sometimes finish faster and end up cheaper overall, despite the higher per-second rate.

### One-line office answer
> Lambda cost mainly depends on requests, execution duration, and memory. Provisioned Concurrency and extra ephemeral storage add charges — Reserved Concurrency itself does not.

---

## 6. Side-by-Side Comparison

| | Reserved Concurrency | Provisioned Concurrency |
|---|---|---|
| **Purpose** | Guarantee + limit capacity | Eliminate cold starts |
| **Cost** | Free | Paid (extra) |
| **Solves cold starts?** | No | Yes |
| **Prevents noisy-neighbor issue?** | Yes | No (that's Reserved's job) |
| **Needs version/alias?** | No | Yes |
| **Can be 0?** | Yes (kills function) | No |
| **Works alone?** | Yes | Yes, but pulls from shared pool unless Reserved is also set |
| **Max value** | Up to account concurrency limit | Unreserved concurrency − 100 |

---

## 7. When to Use What

| Scenario | Use |
|---|---|
| Function talks to a database with limited connections | **Reserved Concurrency** |
| Function must never be starved by other functions | **Reserved Concurrency** |
| User-facing API needs consistent low latency (no cold start) | **Provisioned Concurrency** |
| Both — critical + latency-sensitive (e.g., payment API) | **Reserved + Provisioned together** |
| Want to completely disable a function temporarily | **Reserved Concurrency = 0** |
| Cost-sensitive, cold starts are acceptable | **Neither — use default (unreserved) pool** |

---

## 8. Common Mistakes (Troubleshooting)

| Symptom | Likely Cause | Fix |
|---|---|---|
| `TooManyRequestsException` (429) | Reserved concurrency too low, or account pool exhausted | Increase reserved concurrency or request account limit increase |
| Provisioned concurrency config fails | Trying to apply it on `$LATEST` | Publish a version + alias first |
| Provisioned concurrency stuck `IN_PROGRESS` | Normal — allocation takes a minute or two | Wait and re-check with `get-provisioned-concurrency-config` |
| Other functions suddenly throttling | One function's Reserved/Provisioned concurrency is eating the shared pool | Review total reserved+provisioned vs account limit (1,000 default) |
| Provisioned concurrency "wasted" (low utilization) | Over-provisioned for actual traffic | Monitor `ProvisionedConcurrencyUtilization` in CloudWatch, scale down |

---

## 9. One-Line Summary

> **Reserved Concurrency** = "How many can run, guaranteed and capped."
> **Provisioned Concurrency** = "How many are kept warm and ready, instantly."
