# Complete Guide: EC2 Savings Plans Calculation & Cost Optimization for 9-Hour Daily Operations

---

## Chapter 1 — Introduction & Cost Problem

### 1.1 The 9-Hour Daily Scenario

You have an EC2 instance (or multiple instances) that runs **9 hours per day**:
- Start: 9:00 AM
- Stop: 6:00 PM
- Daily uptime: 9 hours
- Monthly uptime: ~195 hours (9 hours × 21–22 working days)

**The challenge:** Without optimization, you're paying On-Demand rates for these 9 hours, which can be expensive at scale.

**The opportunity:** EC2 Savings Plans can reduce your costs by **20–72%** if configured correctly.

### 1.2 Why This Matters

| Scenario | Monthly Hours | On-Demand Cost | With Savings Plan | Monthly Savings |
|----------|---|---|---|---|
| t3.medium | 195 | $85.22 | $68.38 | $16.84 (19.8%) |
| m5.large | 195 | $170.44 | $118.11 | $52.33 (30.7%) |
| c5.2xlarge | 195 | $682.08 | $409.30 | $272.78 (40%) |
| r5.4xlarge | 195 | $1,364.16 | $697.33 | $666.83 (48.9%) |

**Annual savings:** For a single `c5.2xlarge` instance: **~$3,273 per year**

For 10 instances: **~$32,730 per year**

### 1.3 Common Misconceptions

❌ **"Savings Plans are only for 24/7 servers"**
✅ True: Savings Plans work best for predictable usage. Even 9-hour daily usage is predictable and saves money.

❌ **"I need to commit to a full year to save"**
✅ False: You can choose 1-year or 3-year commitments. 1-year is lower risk.

❌ **"Savings Plans lock me into specific instances"**
✅ Partially true: Some Savings Plans (Compute) are flexible; others (EC2 Instance) are specific.

❌ **"Reserved Instances are the same as Savings Plans"**
✅ False: They're different products with different pricing and flexibility.

---

## Chapter 2 — EC2 Pricing Models Explained

### 2.1 The Five EC2 Pricing Options

AWS offers five ways to pay for EC2 instances:

#### Option 1: On-Demand Pricing

**What it is:** Pay-as-you-go. You're charged hourly (or by the second) for each instance running.

**Best for:** Irregular usage, spikes, testing

**Example (t3.medium in us-east-1):** $0.0416 per hour

**Calculation for 9-hour daily operation:**
```
Hourly rate: $0.0416
Daily cost (9 hours): $0.0416 × 9 = $0.3744
Monthly cost (195 hours): $0.0416 × 195 = $8.112
Annual cost (2,340 hours): $0.0416 × 2,340 = $97.344
```

**Pros:** No upfront cost, flexible, pay exactly for what you use
**Cons:** Expensive, no discount for predictable usage

---

#### Option 2: Savings Plans (Recommended for 9-hour scenario)

**What it is:** Commit to a certain amount of compute (measured in $/hour) for 1 or 3 years, get discounted rate.

**Best for:** Predictable, regular usage (like 9 hours daily)

**Types:**
- **Compute Savings Plan:** Flexible across instance families
- **EC2 Instance Savings Plan:** Specific to instance type/OS

**Example (Compute Savings Plan, 1-year, all upfront):**
- Commitment: $0.0333/hour (20% discount vs. On-Demand $0.0416/hour)
- Term: 1 year
- Payment: $291.48 upfront (365 days × 24 hours × $0.0333)
- For 9-hour daily usage: You're only using ~37.5% of your commitment
- Remaining capacity: Can be used for other instances (flexibility)

**Pros:** 20–40% discount, flexible (Compute plans), no long commitment (1-year option)
**Cons:** Upfront cost, lose discount if usage drops dramatically

---

#### Option 3: Reserved Instances (RI)

**What it is:** Reserve capacity for a specific instance type for 1 or 3 years.

**Best for:** Predictable, specific workloads

**Example (1-year, all upfront for m5.large):** ~$799/year instead of $1,490 On-Demand

**Pros:** Up to 72% discount, guaranteed capacity
**Cons:** Less flexible, instance-specific

---

#### Option 4: Spot Instances

**What it is:** Bid for spare AWS capacity at 40–90% discount, but AWS can terminate anytime.

**Best for:** Fault-tolerant, flexible workloads

**Not recommended for:** 9-hour critical daily workloads (reliability concern)

---

#### Option 5: Dedicated Hosts

**What it is:** Rent an entire physical server for licensing/compliance.

**Cost:** Highest (not recommended for cost optimization)

---

### 2.2 Pricing Comparison Matrix

| Aspect | On-Demand | Savings Plan (1Y) | Savings Plan (3Y) | Reserved Instance (1Y) | Spot |
|--------|-----------|------------------|-------------------|----------------------|------|
| **Hourly Rate** | Full price | 20–40% discount | 30–50% discount | 30–60% discount | 40–90% discount |
| **Upfront Cost** | None | Optional | Optional | Optional | None |
| **Commitment Term** | Hourly | 1 year | 3 year | 1 year | None |
| **Flexibility** | Full | High (Compute) | High (Compute) | Low | None |
| **Capacity Guarantee** | Yes | No | No | Yes | No |
| **For 9-hour daily** | ❌ Expensive | ✅ **Recommended** | ⚠️ High upfront | ⚠️ Inflexible | ❌ Risky |

---

## Chapter 3 — Savings Plans Deep Dive

### 3.1 What Exactly Is a Savings Plan?

A Savings Plan is a commitment to spend a certain amount per hour on AWS compute services (EC2, Fargate, Lambda) for 1 or 3 years, in exchange for a discounted rate.

**Key concept:** You commit to **$/hour of compute**, not to a specific number of instances.

### 3.2 Two Types of Savings Plans

#### Type A: Compute Savings Plan

**Flexibility:** Maximum
**Coverage:** Any instance type, size, region, OS (Linux/Windows), tenancy

**How it works:**
```
You commit: $1.00/hour for 1 year
You can use it on:
  - 1 × t3.large ($1.00/hour) OR
  - 4 × t3.small ($0.25/hour each) OR
  - 1 × m5.large ($0.70/hour) + 2 × t3.small ($0.15/hour) each
  - OR different instances on different days
  - OR different regions
  
Flexibility: Highest. If you don't use your full commitment,
no problem — use it on other instances.
```

**Best for:** Organizations with variable workloads, multiple instance types

**Discount:** 20–40% off On-Demand (varies by region, term, payment)

---

#### Type B: EC2 Instance Savings Plan

**Flexibility:** Lower than Compute Plans
**Coverage:** Specific instance type + size + region + OS

**How it works:**
```
You commit: $0.50/hour for m5.large Linux in us-east-1 for 1 year
You can use it on:
  - Any number of m5.large Linux instances in us-east-1
  - Only m5.large (not m5.xlarge, not m5.large Windows)
  
Flexibility: Lower. Locked to instance type. But slightly cheaper
than Compute Savings Plan.
```

**Discount:** 20–45% off On-Demand (often 5–10% more than Compute plans)

---

### 3.3 Payment Options

#### All Upfront
- Pay entire year's cost (or 3-year cost) upfront
- Discount: Maximum (typically 35–40% for 1-year Compute plans)
- Example: $291.48 for entire year, save on each use

#### Partial Upfront
- Pay part upfront, rest monthly
- Discount: Moderate (25–35%)
- Example: $145.74 upfront + $24.29/month

#### No Upfront
- Pay monthly, no upfront cost
- Discount: Lowest (10–20%)
- Example: $24.29/month

---

### 3.4 Why Savings Plans Beat Reserved Instances

| Feature | Savings Plan | Reserved Instance |
|---------|-------------|-------------------|
| **Flexibility** | High (any instance) | Low (specific instance) |
| **Discount** | 20–40% | 30–60% (but specific) |
| **Coverage** | EC2, Fargate, Lambda | EC2 only |
| **Capacity guarantee** | No | Yes |
| **For 9-hour daily** | ✅ Perfect | ⚠️ Overkill (wasted commitment) |

---

## Chapter 4 — 9-Hour Daily Operations Analysis

### 4.1 Why 9 Hours Matters

**9 hours per day = 195 hours per month (22 working days)**

This is the **optimal range** for Savings Plans because:
- You have predictable usage ✅
- You're not paying for unused hours 24/7 ✅
- Savings Plans discount is still significant ✅

### 4.2 Monthly Hours Calculation

| Scenario | Days/Week | Hours/Day | Weeks/Month | Monthly Hours |
|----------|-----------|-----------|-------------|---------------|
| Business days only | 5 | 9 | 4.3 | 194.7 ≈ 195 |
| Every day | 7 | 9 | 4.3 | 270.9 ≈ 271 |
| Business days + weekend | 6 | 9 | 4.3 | 233.4 ≈ 233 |

**For our example:** We'll use **195 hours/month** (business days only, most common)

### 4.3 Annual Hours from 9-Hour Daily Usage

```
9 hours/day × 250 working days/year = 2,250 hours/year

Alternative calculations:
  - 9 hours × 365 days = 3,285 hours/year (24/7 equivalent is 8,760)
  - 9 hours × 252 business days = 2,268 hours/year
  - 9 hours × 261 days (avg with holidays) = 2,349 hours/year
```

**We'll use 2,250–2,300 hours/year for calculations**

---

## Chapter 5 — Cost Calculation Methods

### 5.1 Method 1: Direct Hourly Calculation

**Formula:**
```
Monthly Cost = Hourly Rate × Monthly Hours
Annual Cost = Hourly Rate × Annual Hours
```

**Example: t3.medium in us-east-1**

| Metric | Value |
|--------|-------|
| On-Demand hourly rate | $0.0416 |
| Monthly hours (9/day, 22 days) | 198 |
| Monthly On-Demand cost | $0.0416 × 198 = $8.24 |
| Annual On-Demand cost | $0.0416 × 2,376 = $98.92 |

**With Savings Plan (1-year, all upfront, Compute):**

| Metric | Value |
|--------|-------|
| Savings Plan hourly rate | $0.0333 (20% discount) |
| Annual commitment | $0.0333 × 8,760 hours = $291.48 |
| You use per year | 2,376 hours |
| Effective cost for your usage | $0.0333 × 2,376 = $79.10 |
| Annual savings | $98.92 - $79.10 = $19.82 |
| Savings percentage | 20% |

⚠️ **Note:** You're wasting $212.38/year (the portion you don't use)

---

### 5.2 Method 2: Blended Hourly Rate

When you don't use your full Savings Plan commitment, calculate the **actual cost per hour**:

**Formula:**
```
Blended Hourly Rate = Savings Plan Annual Cost / Annual Hours You Use
```

**Example:**
```
Savings Plan cost (1-year all upfront): $291.48 per year
Hours you actually use: 2,376
Blended hourly rate: $291.48 / 2,376 = $0.1227

This is your TRUE cost per hour (higher than promised $0.0333
because you're only using 27% of your commitment)
```

**Comparison:**
- On-Demand: $0.0416/hour
- Savings Plan: $0.0333/hour (promised rate)
- **Blended rate: $0.1227/hour** (YOUR actual cost including wasted commitment)

**This is why Savings Plans aren't ideal for 9-hour daily usage of a single instance!**

---

### 5.3 Method 3: Cost Per Day Calculation

**Formula:**
```
Daily Cost = Hourly Rate × 9 hours
```

**Example: m5.large in us-east-1**

| Pricing Model | Hourly Rate | Daily Cost (9h) | Monthly Cost | Annual Cost |
|---|---|---|---|---|
| On-Demand | $0.0960 | $0.864 | $16.85 | $207.20 |
| Savings Plan (1Y, all up) | $0.0672 | $0.605 | $11.79 | $75.59 |
| Savings Plan blended (1Y) | $0.1599 | $1.439 | $28.06 | $328 (including waste) |
| Reserved Instance (1Y, all up) | $0.0576 | $0.518 | $10.10 | $80.41 |

---

## Chapter 6 — Real-World Cost Scenarios

### Scenario 1: Single t3.medium Instance, 9 hours daily

**Assumptions:**
- Region: us-east-1 (N. Virginia)
- Instance: t3.medium
- Usage: 9 AM–6 PM, Monday–Friday
- Duration: 1 year

**Pricing data (as of June 2026):**
- On-Demand: $0.0416/hour
- Savings Plan (Compute, 1Y, all up): $0.0333/hour base
- Monthly working hours: 195

**Cost Breakdown:**

| Model | Monthly | Annual | Savings vs. On-Demand |
|-------|---------|--------|----------------------|
| **On-Demand** | $8.11 | $97.34 | — |
| **Savings Plan (pay all up)** | $0 initial + $6.50/mo | $77.86 (after $0 setup) | **$19.48 (20%)** |
| **Savings Plan (blended with waste)** | $13.02 | $155.87 | ❌ **60% more expensive!** |

**Insight:** Using Savings Plans for a single small instance wastes money because the $291.48 commitment is way more than needed.

---

### Scenario 2: 10 × m5.large Instances, 9 hours daily

**Assumptions:**
- 10 instances (horizontal scaling)
- Each runs 9 hours/day
- Monthly hours: 195 per instance = 1,950 total hours

**Pricing data:**
- On-Demand: $0.0960/hour per instance
- Savings Plan (Compute, 1Y, all up): $0.0672/hour per instance

**Cost Breakdown:**

| Model | Monthly | Annual | Savings |
|-------|---------|--------|----------|
| **On-Demand** | $187.20 | $2,246.40 | — |
| **Savings Plan (all up)** | $131.04 | $1,572.48 | **$673.92 (30%)** |
| **Savings Plan (no waste)** | Same | Same | ✅ **Clean savings!** |

**Key difference:** With 10 instances, you're using most of your Savings Plan commitment. No waste!

---

### Scenario 3: Mixed Workload (3 instance types, 9 hours daily)

**Assumptions:**
- 2 × t3.medium
- 3 × m5.large
- 1 × c5.2xlarge
- All run 9 hours/day, 195 hours/month

**Hourly costs (On-Demand, us-east-1):**
- t3.medium: $0.0416 × 2 = $0.0832/hour
- m5.large: $0.0960 × 3 = $0.2880/hour
- c5.2xlarge: $0.3400 × 1 = $0.3400/hour
- **Total: $0.7112/hour combined**

**Monthly costs:**

| Model | Monthly Cost |
|-------|---|
| On-Demand | $0.7112 × 195 = $138.68 |
| Savings Plan (Compute, 1Y, all up, 25% discount) | $0.5334 × 195 = $104.01 |
| Savings vs. On-Demand | **$34.67 (25%)** |

---

## Chapter 7 — Savings Plan vs. On-Demand vs. Reserved Instances

### Comprehensive Comparison

| Factor | On-Demand | Savings Plan (1Y) | Savings Plan (3Y) | Reserved Instance | Spot |
|--------|-----------|-------------------|-------------------|-------------------|------|
| **Monthly cost (t3.medium, 195h)** | $8.11 | $6.50 | $5.46 | $6.30 | $2–$4 |
| **Upfront cost** | None | $77.86 | $167.01 | $75.60 | None |
| **Total annual cost (1Y)** | $97.34 | $77.86 | N/A | $75.60 | $24–$48 |
| **Total 3-year cost** | $292.02 | $233.58 | $167.01 | $226.80 | $72–$144 |
| **Flexibility** | Max | High | High | Low | None |
| **Commitment risk** | None | 1 year | 3 years | 1 year | None |
| **For 9-hour daily** | ❌ Expensive | ✅ **BEST** | ⚠️ High upfront | ⚠️ Overkill | ❌ Risky |

### Decision Matrix for 9-Hour Usage

```
Are you running 9 hours daily?
  └─ YES
      ├─ Predictable schedule (same 9 hours daily)?
      │  ├─ YES
      │  │  ├─ Single instance?
      │  │  │  ├─ YES → Use On-Demand (Savings Plan overhead not worth it)
      │  │  │  └─ NO (multiple instances) → Savings Plan (1Y, all up) ✅
      │  │  └─ Multiple instance types?
      │  │     ├─ YES → Compute Savings Plan ✅
      │  │     └─ NO → EC2 Instance Savings Plan ✅
      │  └─ NO (variable schedule) → On-Demand or 1Y Compute Savings Plan ⚠️
      └─ NO
          └─ Use On-Demand or Spot
```

---

## Chapter 8 — Hands-on Lab: Calculate Your Costs

### 8.1 Lab Overview

You'll calculate costs for your specific scenario using AWS Pricing Calculator.

**Time:** 30–45 minutes
**What you'll learn:** How to use AWS tools to find real pricing, build custom cost models

---

### 8.2 Method 1: AWS Pricing Calculator (Easiest)

**Step 1: Open AWS Pricing Calculator**
1. Go to: https://calculator.aws
2. Click "Create estimate"

**Step 2: Add EC2 Instances**
1. Search for "EC2"
2. Click "Configure"

**Step 3: Configure Your Instance**

| Setting | Value |
|---------|-------|
| Region | us-east-1 |
| Operating system | Linux |
| Workload | Daily usage (I want to use Savings Plans) |
| Quantity | 1 (start with single instance) |
| Instance type | t3.medium |
| Pricing strategy | Savings Plans |
| Savings Plan term | 1 year |
| Payment option | All Upfront |
| Instance hours per month | 195 |
| Storage | 30 GB gp3 |

**Step 4: View Results**
- See monthly cost
- See annual cost
- See Savings Plan discount percentage

**Step 5: Compare by changing "Pricing strategy" to:**
- On-Demand (see higher cost)
- Reserved Instance (1 year)
- Spot (see lower cost)

---

### 8.2 Method 2: Manual Calculation (Spreadsheet)

Create a spreadsheet in Excel or Google Sheets:

**Headers:**
- Instance Type
- Region
- On-Demand Hourly Rate
- Monthly Hours (195)
- Monthly On-Demand Cost
- Savings Plan 1-Year Hourly Rate
- Annual Savings Plan Cost (All Upfront)
- Effective Monthly Cost (with commitment)
- Monthly Savings vs. On-Demand

**Example rows:**

```
Instance Type | Region | On-Demand/hr | Monthly Hours | Monthly Cost
t3.medium     | us-e-1 | $0.0416      | 195           | $8.11
m5.large      | us-e-1 | $0.0960      | 195           | $18.72
c5.2xlarge    | us-e-1 | $0.3400      | 195           | $66.30
r5.4xlarge    | us-e-1 | $1.0896      | 195           | $212.47

Savings Plan costs (1-year, all up, Compute plans):
Instance Type | Savings Plan $/hr | Annual Cost | Monthly Effective
t3.medium     | $0.0333           | $291.48     | $24.29
m5.large      | $0.0672           | $583.00     | $48.58
```

---

### 8.3 Method 3: AWS CLI Pricing (Advanced)

```bash
# Get EC2 pricing using AWS Pricing API
# (Note: AWS doesn't offer a direct CLI for pricing, but you can use APIs)

# Example: Query pricing from JSON data file
python3 << 'PYTHON'
import json
import urllib.request

# Fetch pricing data from AWS
url = "https://pricing.aws.amazon.com/pricing/file/aws-pricing-index.json"
response = urllib.request.urlopen(url)
pricing_index = json.loads(response.read())

# Find EC2 pricing for a specific region
region = "us-east-1"
instance_type = "t3.medium"

# Parse data (simplified example)
# In reality, pricing JSON is complex and requires careful parsing
print(f"Pricing for {instance_type} in {region}")

PYTHON
```

---

## Chapter 9 — Optimization Strategies

### 9.1 Strategy 1: Single Instance + On-Demand (Simplest)

**When to use:** 1–2 instances, 9 hours daily

**Cost:** On-Demand rate × 195 hours/month

**Example (t3.medium):**
- Monthly: $8.11
- Annual: $97.34
- Complexity: Minimal (automatic)
- Savings: $0

**Pros:** Simple, no commitment, flexible
**Cons:** No cost savings

---

### 9.2 Strategy 2: Multiple Instances + Compute Savings Plan (Recommended)

**When to use:** 3+ instances, predictable 9-hour schedule

**Cost:** Savings Plan hourly rate × 8,760 hours/year (committed)

**Example (10 × m5.large):**
- Savings Plan cost: $1,572.48/year (1-year, all up)
- You use: 1,950 hours/year (10 instances × 195 hours)
- Actual hourly cost: $1,572.48 / 1,950 = $0.807/hour
- On-Demand equivalent: $0.0960 × 10 × 195 = $187.20/month
- Savings Plan equivalent: $131.04/month
- **Annual savings: $673.92 (30%)**

**Pros:** 30% savings, flexible across instance types, can use excess capacity
**Cons:** Upfront cost, 1-year commitment

---

### 9.3 Strategy 3: Savings Plan + Spot for Flexible Capacity

**When to use:** 9-hour core capacity + occasional bursting needs

**Setup:**
- Commit to Savings Plan for core instances (9 hours daily)
- Use Spot instances for additional capacity when needed (no commitment)

**Example:**
- Core: 5 × m5.large on Savings Plan = $789/year
- Flexible: Additional m5.large via Spot = $50–$200/year (variable)
- **Total flexibility with most savings!**

**Pros:** Best price flexibility, can scale up easily
**Cons:** More complex management

---

### 9.4 Strategy 4: Rightsizing to Smaller Instance Type

**When to use:** Current instance is over-sized

**Example:**
- Running c5.2xlarge but only using 30% CPU/Memory
- Downsize to c5.large: **Cuts cost in half**

```
Current:       c5.2xlarge → $0.34/hour → $66.30/month (195h)
Rightsized:    c5.large   → $0.085/hour → $16.58/month (195h)
Savings: $49.72/month = $596.64/year!
```

**Action plan:**
1. Monitor CloudWatch CPU and Memory for 2 weeks
2. Identify if current instance is over-provisioned
3. Test smaller instance with your workload
4. Switch if it performs adequately

---

### 9.5 Strategy 5: Consolidate into Fewer, Larger Instances

**When to use:** Many small instances

**Example:**
- Current: 10 × t3.small = $0.024 × 10 × 195 = $46.80/month
- Consolidated: 1 × t3.xlarge = $0.166 × 195 = $32.37/month
- **Savings: $14.43/month = $173/year**

**Considerations:**
- Verify single large instance can handle load
- Reduces redundancy (high-availability concern)
- Better for non-critical workloads

---

## Chapter 10 — Hourly Breakdown & Blended Rates

### 10.1 Hour-by-Hour Cost Breakdown (1 Day)

**Scenario: m5.large, 9:00 AM – 6:00 PM, Savings Plan $0.0672/hour**

| Hour | Start | End | Running? | Cost per hour | Cumulative |
|------|-------|-----|----------|---|---|
| 1 | 9:00 AM | 10:00 AM | Yes | $0.0672 | $0.0672 |
| 2 | 10:00 AM | 11:00 AM | Yes | $0.0672 | $0.1344 |
| ... | ... | ... | ... | $0.0672 | ... |
| 9 | 5:00 PM | 6:00 PM | Yes | $0.0672 | $0.6048 |
| 10 | 6:00 PM | 7:00 PM | No | $0.00 | $0.6048 |
| ... | ... | ... | No | $0.00 | $0.6048 |
| 24 | 8:00 AM | 9:00 AM | No | $0.00 | **$0.6048** |

**Daily cost: $0.6048**

---

### 10.2 Month-by-Month Savings Progression

**Setup: Compute Savings Plan 1-year, all upfront commitment = $583 (for m5.large)**

| Month | Hours Used | Cost | Committed | Remaining Budget | Utilization % |
|-------|-----------|------|-----------|------------------|---|
| 1 (Jan) | 195 | $13.10 | $583.00 | $569.90 | 33.4% |
| 2 (Feb) | 195 | $13.10 | $583.00 | $557.00 | 33.4% |
| 3 (Mar) | 195 | $13.10 | $583.00 | $543.90 | 33.4% |
| ... | ... | ... | ... | ... | ... |
| 12 (Dec) | 195 | $13.10 | $583.00 | $0 | 100% (year complete) |

**Annual totals:**
- Used: 195 × 12 = 2,340 hours
- Cost: $13.10 × 12 = $157.20
- Actual rate: $157.20 / 2,340 = **$0.067/hour** (matches promised rate)
- On-Demand equivalent: $0.096 × 2,340 = $224.64
- **Annual savings: $67.44 (30%)**

---

### 10.3 Blended Rate Calculation

**For single m5.large instance, 9 hours daily:**

```
Savings Plan annual cost (1Y all up): $583.00
Actual hours per year: 2,340 hours (9 hours × 260 working days)

Blended hourly rate = $583.00 / 2,340 = $0.249/hour

This is higher than the promised $0.0672/hour because you're
not using your full Savings Plan commitment (only using $157.20
of the $583 annual commitment).

It's WORSE than On-Demand ($0.0960/hour) for a single instance!

INSIGHT: Savings Plans only make sense if you can:
  1. Use the full commitment across multiple instances, OR
  2. Use the excess capacity for other workloads
```

---

## Chapter 11 — Interview Questions

### 11.1 Beginner Questions (15)

1. **Q: What is On-Demand pricing for EC2?**
   A: Pay-as-you-go hourly pricing. You're charged for each hour an instance runs, with no upfront commitment.

2. **Q: What are AWS Savings Plans?**
   A: Pricing model where you commit to spending a certain amount per hour (on compute) for 1 or 3 years, getting a discount in exchange.

3. **Q: What discount do Savings Plans offer?**
   A: 20–40% off On-Demand pricing for 1-year, 30–50% off for 3-year plans.

4. **Q: What is the difference between Compute Savings Plans and EC2 Instance Savings Plans?**
   A: Compute plans are flexible across instance types; EC2 plans are specific to one instance type.

5. **Q: For 9-hour daily usage, which is better: On-Demand or Savings Plans?**
   A: Depends on quantity. Single instance: On-Demand. Multiple instances: Savings Plans.

6. **Q: What is "blended hourly rate"?**
   A: Your actual cost per hour including wasted commitment. Can be higher than On-Demand for single small instances.

7. **Q: How many hours per month is 9 hours daily?**
   A: Approximately 195 hours/month (9 × 22 working days).

8. **Q: Can you cancel a Savings Plan early?**
   A: No. You're committed for 1 or 3 years, but you can sell it on AWS Marketplace.

9. **Q: What is a Reserved Instance (RI)?**
   A: Similar to Savings Plans but specific to an instance type. Offers up to 72% discount but less flexibility.

10. **Q: Does a Savings Plan lock you into a specific region?**
    A: No. Compute Savings Plans work across regions (you specify at purchase). EC2 Instance plans are region-specific.

11. **Q: What is Spot Pricing?**
    A: Heavily discounted (40–90% off) spare AWS capacity, but AWS can terminate instances anytime.

12. **Q: Is Spot suitable for a 9-hour daily workload?**
    A: Not ideal. Spot is for flexible, fault-tolerant workloads. 9-hour daily usually needs reliability.

13. **Q: What does "all upfront" payment mean?**
    A: You pay the entire year's or 3-year's cost upfront. Gives maximum discount.

14. **Q: Can multiple instances share a Savings Plan?**
    A: Yes! You commit to $/hour, which can be distributed across any instances you want.

15. **Q: If I don't use my full Savings Plan commitment, do I lose money?**
    A: Yes, you've paid upfront. That's why Savings Plans are best for predictable, high usage.

### 11.2 Intermediate Questions (10)

1. **Q: Calculate the monthly cost for 3 × m5.large instances at 9 hours daily with Savings Plans.**
   A: 
   - On-Demand rate: $0.0960/hour
   - Monthly hours: 195
   - On-Demand monthly: $0.0960 × 3 × 195 = $56.16
   - Savings Plan (25% discount): $56.16 × 0.75 = $42.12/month
   - Monthly savings: $14.04 (25%)

2. **Q: What is the blended hourly rate for a single t3.medium with Compute Savings Plan (1Y, all up)?**
   A:
   - Savings Plan cost: $291.48/year (1-year term, all upfront)
   - Actual usage: 2,340 hours/year (9h/day × 260 days)
   - Blended rate: $291.48 / 2,340 = $0.1246/hour
   - This is much higher than promised $0.0333/hour

3. **Q: Compare total 3-year costs: On-Demand vs. 1-Year SP (renew yearly) vs. 3-Year SP**
   A: For m5.large at 195h/month:
   - On-Demand: $0.0960 × 2,340h/year × 3 = $674.88
   - 1Y SP renewed 3x: $449.28 × 3 = $1,347.84
   - 3Y SP (all up): $783.00 (single upfront commitment)
   (All upfront 3Y is best if you're sure you'll use it)

4. **Q: Why are Savings Plans better than Reserved Instances for 9-hour usage?**
   A: RIs are instance-specific and don't accommodate variable usage well. Savings Plans are flexible across instance types and can use excess capacity for other workloads.

5. **Q: How would you optimize costs for a workload that runs 9 hours daily but has occasional 24-hour peaks?**
   A: Use Savings Plans for the 9-hour baseline, Spot instances for burst capacity. This gives stable costs for core + flexibility for peak demand.

6. **Q: Calculate cost for mixed workload: 2 × t3.medium + 3 × m5.large, 9h daily, Savings Plan (1Y)**
   A:
   - t3.medium: $0.0416/h × 2 = $0.0832/h
   - m5.large: $0.0960/h × 3 = $0.2880/h
   - Total: $0.3712/h = $72.38/month On-Demand
   - With 25% Savings Plan discount: $54.29/month
   - Annual savings: $216.54

7. **Q: Why does a Savings Plan cost more than On-Demand when you only use 27% of the commitment?**
   A: You paid upfront for 100% capacity but only use 27%. Your effective hourly rate includes the wasted commitment, making it higher than On-Demand hourly rate.

8. **Q: For a 9-hour daily workload, what's the minimum number of instances where Savings Plans make sense?**
   A: Approximately 3–5 instances of m5.large class. Below that, On-Demand is cheaper because Savings Plan overhead isn't justified.

9. **Q: How would you calculate cost for a workload running 9 hours daily in two regions (us-east-1 and eu-west-1)?**
   A: 
   - Each region gets its own Savings Plan commitment
   - Calculate separately: (rate₁ × h × days) + (rate₂ × h × days)
   - Compute Savings Plans can cover both regions with single commitment

10. **Q: If you downsize an instance from c5.2xlarge to c5.large, how much would costs decrease?**
    A:
    - c5.2xlarge: $0.34/hour
    - c5.large: $0.085/hour
    - Savings: ($0.34 - $0.085) × 195h/month = $49.73/month
    - Annual: ~$597

---

## Chapter 12 — Cost Calculator Tools

### 12.1 Simple Spreadsheet Calculator

**Create this in Google Sheets or Excel:**

```
HEADERS:
A1: Instance Type
B1: On-Demand $/hr
C1: Savings Plan $/hr (1Y)
D1: Monthly Hours
E1: Monthly Cost (OD)
F1: Monthly Cost (SP)
G1: Monthly Savings
H1: Annual Savings

EXAMPLE DATA:
A2: t3.medium
B2: =0.0416
C2: =0.0333
D2: =195
E2: =B2*D2
F2: =C2*D2
G2: =E2-F2
H2: =G2*12

TOTALS:
E10: =SUM(E2:E9)
F10: =SUM(F2:F9)
G10: =SUM(G2:G9)
H10: =SUM(H2:H9)
```

---

### 12.2 Advanced Python Calculator

```python
#!/usr/bin/env python3
"""
EC2 Savings Plan Cost Calculator for 9-hour Daily Usage
"""

class EC2CostCalculator:
    def __init__(self, on_demand_hourly, sp_hourly, monthly_hours=195):
        self.on_demand_hourly = on_demand_hourly
        self.sp_hourly = sp_hourly
        self.monthly_hours = monthly_hours
    
    def calculate_on_demand(self):
        monthly = self.on_demand_hourly * self.monthly_hours
        annual = monthly * 12
        return {"monthly": monthly, "annual": annual}
    
    def calculate_savings_plan(self, upfront_annual, annual_hours=2340):
        """
        Calculate Savings Plan costs considering upfront payment and usage
        """
        monthly_effective = upfront_annual / 12
        blended_hourly = upfront_annual / annual_hours
        
        return {
            "upfront": upfront_annual,
            "monthly_effective": monthly_effective,
            "blended_hourly": blended_hourly,
            "annual_used": self.sp_hourly * annual_hours
        }
    
    def compare(self):
        od = self.calculate_on_demand()
        sp_commitment = self.sp_hourly * 8760  # Full year commitment
        sp = self.calculate_savings_plan(sp_commitment)
        
        print(f"\n{'='*60}")
        print(f"EC2 Cost Comparison")
        print(f"{'='*60}")
        print(f"\nON-DEMAND:")
        print(f"  Hourly rate: ${self.on_demand_hourly:.4f}")
        print(f"  Monthly (195h): ${od['monthly']:.2f}")
        print(f"  Annual: ${od['annual']:.2f}")
        
        print(f"\nSAVINGS PLAN (1-Year, All Upfront):")
        print(f"  Upfront cost: ${sp['upfront']:.2f}")
        print(f"  Monthly equivalent: ${sp['monthly_effective']:.2f}")
        print(f"  Blended hourly: ${sp['blended_hourly']:.4f}")
        print(f"  Your actual annual cost (at 2340h): ${sp['annual_used']:.2f}")
        
        print(f"\nSAVINGS:")
        savings_annual = od['annual'] - sp['annual_used']
        print(f"  Annual savings: ${savings_annual:.2f}")
        print(f"  Percentage: {(savings_annual/od['annual']*100):.1f}%")
        
        # Blended analysis (with waste)
        print(f"\nBLENDED RATE ANALYSIS (including wasted capacity):")
        print(f"  You commit to: ${sp['upfront']:.2f}/year")
        print(f"  You use: 2,340 hours/year")
        print(f"  Wasted: {(1 - 2340/8760)*100:.1f}% of commitment")
        print(f"  Effective cost: ${sp['upfront']/2340:.4f}/hour")
        
        # Is it worth it?
        if savings_annual > 0 and sp['blended_hourly'] < self.on_demand_hourly:
            print(f"\n✅ Savings Plans ARE worth it for this workload!")
        else:
            print(f"\n❌ Savings Plans may NOT be worth it (too much wasted capacity)")

# Example usage
if __name__ == "__main__":
    # Single m5.large instance
    print("SCENARIO 1: Single m5.large")
    calc = EC2CostCalculator(
        on_demand_hourly=0.0960,
        sp_hourly=0.0672
    )
    calc.compare()
    
    # Multiple instances
    print("\n\nSCENARIO 2: 10 × m5.large instances (shared Savings Plan)")
    calc_multi = EC2CostCalculator(
        on_demand_hourly=0.0960 * 10,
        sp_hourly=0.0672 * 10
    )
    calc_multi.compare()
```

**Run it:**
```bash
python3 ec2_cost_calculator.py
```

---

## Chapter 13 — Summary & Quick Reference

### 13.1 Key Takeaways (10 Points)

1. **9-hour daily usage = 195 hours/month, 2,340 hours/year** — Predictable pattern perfect for optimization.

2. **On-Demand is fine for single small instances** — The overhead of Savings Plans isn't worth it unless you have 3+ instances.

3. **Compute Savings Plans beat EC2 Instance Savings Plans** — More flexibility across instance types.

4. **Watch out for "blended rates"** — If you only use 27% of your commitment, your effective rate can be worse than On-Demand.

5. **1-year Savings Plans are better than 3-year for risk mitigation** — 3-year is cheaper but locks you in for 3 years.

6. **All Upfront payment gives max discount** — 35–40% off vs. 20–25% off for partial/no upfront.

7. **Rightsize instances before committing** — A c5.large on Savings Plan is cheaper than c5.2xlarge On-Demand.

8. **Use Spot for burst capacity** — Combine Savings Plans (baseline) + Spot (flexible) for best price.

9. **Consolidate instances when possible** — 10 × t3.small costs more than 1 × t3.large with better resource efficiency.

10. **Monitor and recalculate quarterly** — Usage patterns change; re-evaluate Savings Plans every 3 months.

---

### 13.2 Quick Reference Table

```
┌─────────────────────────────────────────────────────────────────────┐
│         EC2 SAVINGS FOR 9-HOUR DAILY USAGE - QUICK GUIDE            │
└─────────────────────────────────────────────────────────────────────┘

INSTANCE TYPE         ON-DEMAND/hr   SAVINGS PLAN/hr (1Y)   MONTHLY SAVINGS
─────────────────────────────────────────────────────────────────────
t3.medium             $0.0416        $0.0333                $1.65 (single)
t3.large              $0.0832        $0.0666                $3.30 (single)
m5.large              $0.0960        $0.0672                $5.65 (single)
m5.xlarge             $0.1920        $0.1344                $11.31 (single)
c5.large              $0.085         $0.0595                $2.79 (single)
c5.2xlarge            $0.340         $0.238                 $19.95 (single)
r5.2xlarge            $0.504         $0.3528                $18.00 (single)

FOR 10 INSTANCES OF SAME TYPE:
─────────────────────────────────────────────────────────────────────
Instance Type        On-Demand/mo    Savings Plan/mo    Annual Savings
m5.large             $187.20         $131.04            $673.92
c5.2xlarge           $66.30          $46.41             $238.68
r5.2xlarge           $98.28          $68.80             $353.76

DECISION GUIDE:
─────────────────────────────────────────────────────────────────────
Instances    Uptime Pattern         Recommendation
─────────────────────────────────────────────────────────────────────
1–2          9h daily               On-Demand (no overhead)
3–5          9h daily (consistent)  1-Year Compute SP (all up)
5+           9h daily (consistent)  1-Year Compute SP (all up)
1–2          Variable hours         On-Demand (flexibility)
ANY          Burst capacity         Spot instances
```

---

### 13.3 Cost Formulas Summary

```
MONTHLY COST CALCULATIONS:
─────────────────────────────────────────────
On-Demand:
  Cost = Hourly Rate × 9 hours × 22 working days
       = Hourly Rate × 198 hours (approximate)

Savings Plan (1-Year, All Upfront):
  Annual Commitment = Hourly Rate × 8,760 hours
  Monthly Cost = Annual Commitment / 12
  (Includes full commitment, even if unused)

Effective Cost (actual usage):
  Monthly Effective = (Hourly Rate × 9 × 22)
  Annual Effective = (Hourly Rate × 2,340)

Blended Hourly Rate (with waste):
  Blended = Annual Commitment / Actual Hours Used
  Example: $583 / 2,340h = $0.249/h (worse than OD!)

Monthly Savings:
  Savings = (OD Rate - SP Rate) × Monthly Hours
  Percentage = (Savings / OD Cost) × 100%

ANNUAL SAVINGS:
  = Monthly Savings × 12
```

---

