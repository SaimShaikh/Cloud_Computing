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

### Example
```
Account concurrency limit = 1,000
Reserved concurrency for "order-processor" = 100

Result:
- order-processor can run up to 100 times at once, never more
- Those 100 slots are locked for order-processor only
- Remaining 900 slots are shared by all other functions
```

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

## 4. Side-by-Side Comparison

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

## 5. When to Use What

| Scenario | Use |
|---|---|
| Function talks to a database with limited connections | **Reserved Concurrency** |
| Function must never be starved by other functions | **Reserved Concurrency** |
| User-facing API needs consistent low latency (no cold start) | **Provisioned Concurrency** |
| Both — critical + latency-sensitive (e.g., payment API) | **Reserved + Provisioned together** |
| Want to completely disable a function temporarily | **Reserved Concurrency = 0** |
| Cost-sensitive, cold starts are acceptable | **Neither — use default (unreserved) pool** |

---

## 6. Common Mistakes (Troubleshooting)

| Symptom | Likely Cause | Fix |
|---|---|---|
| `TooManyRequestsException` (429) | Reserved concurrency too low, or account pool exhausted | Increase reserved concurrency or request account limit increase |
| Provisioned concurrency config fails | Trying to apply it on `$LATEST` | Publish a version + alias first |
| Provisioned concurrency stuck `IN_PROGRESS` | Normal — allocation takes a minute or two | Wait and re-check with `get-provisioned-concurrency-config` |
| Other functions suddenly throttling | One function's Reserved/Provisioned concurrency is eating the shared pool | Review total reserved+provisioned vs account limit (1,000 default) |
| Provisioned concurrency "wasted" (low utilization) | Over-provisioned for actual traffic | Monitor `ProvisionedConcurrencyUtilization` in CloudWatch, scale down |

---

## 7. One-Line Summary

> **Reserved Concurrency** = "How many can run, guaranteed and capped."
> **Provisioned Concurrency** = "How many are kept warm and ready, instantly."
