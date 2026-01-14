# AWS Scaling Policies - Complete Guide

## Table of Contents
1. [Introduction](#introduction)
2. [What Are Scaling Policies?](#what-are-scaling-policies)
3. [Core Components](#core-components)
4. [Types of Scaling Policies](#types-of-scaling-policies)
5. [Dynamic Scaling Policies](#dynamic-scaling-policies)
6. [Scheduled Scaling](#scheduled-scaling)
7. [Predictive Scaling](#predictive-scaling)
8. [Best Practices](#best-practices)
9. [Real-World Examples](#real-world-examples)

---

## Introduction

AWS Scaling Policies are a fundamental part of building resilient, cost-effective cloud applications. They allow your infrastructure to automatically adjust resources based on demand, ensuring your application performs well without maintaining excess capacity.

Think of it like an elevator: it only runs when people need it, and the more people waiting, the more elevators operate. Similarly, AWS scaling policies add instances when demand increases and remove them when demand decreases.

---

## What Are Scaling Policies?

A **scaling policy** is a set of rules that tell AWS Auto Scaling when and how to add or remove instances from your Auto Scaling Group (ASG).

### Key Benefits:
- **High Availability**: Ensures your application always has enough resources
- **Cost Optimization**: Removes unused instances during low-traffic periods
- **Automatic Management**: No manual intervention needed
- **Performance**: Maintains consistent application performance

### Scaling Terminology:

| Term | Meaning |
|------|---------|
| **Scale Out** | Adding more instances to handle increased load |
| **Scale In** | Removing instances when demand decreases |
| **Desired Capacity** | The number of instances you want running |
| **Current Capacity** | The actual number of running instances |
| **Min Capacity** | Minimum instances that must always run |
| **Max Capacity** | Maximum instances that can run |

---

## Core Components

Before understanding scaling policies, you need to know these three components:

### 1. Launch Template
Defines how to create new EC2 instances:
- AMI (image)
- Instance type (t2.micro, t3.medium, etc.)
- Security groups
- Key pairs
- User data scripts

### 2. Auto Scaling Group (ASG)
Container that manages a group of instances:
- Sets minimum, maximum, and desired instance counts
- Manages instance lifecycle (creation and termination)
- Coordinates with scaling policies

### 3. Scaling Policies
Rules that determine when and how to scale:
- Monitors CloudWatch metrics
- Triggers scaling actions based on thresholds
- Controls the scaling adjustment size

---

## Types of Scaling Policies

AWS provides four main types of scaling policies:

```
┌─────────────────────────────────────────┐
│    AWS SCALING POLICIES                 │
├─────────────────────────────────────────┤
│                                         │
│  1. DYNAMIC SCALING                     │
│     ├── Target Tracking                 │
│     ├── Step Scaling                    │
│     └── Simple Scaling                  │
│                                         │
│  2. SCHEDULED SCALING                   │
│                                         │
│  3. PREDICTIVE SCALING                  │
│                                         │
└─────────────────────────────────────────┘
```

---

## Dynamic Scaling Policies

Dynamic scaling adjusts resources in real-time based on current demand. It uses CloudWatch alarms to monitor metrics and trigger scaling actions.

### How Dynamic Scaling Works:

```
CloudWatch                  Scaling Policy           Auto Scaling
Monitors Metrics    →    Evaluates Threshold   →   Takes Action
(CPU, Network, etc)      (Is CPU > 80%?)        (Add/Remove Instances)
```

---

### 1. Target Tracking Scaling (Recommended ⭐)

**What it does**: Automatically maintains a specific metric value by adding or removing instances.

**How it works**: Like a thermostat! You set a target temperature (metric value), and the system keeps it at that level.

```
Target: 50% CPU Utilization

Current State:
├── CPU Usage is 75%
├── System: "Too high! Need more instances"
└── Action: Scale OUT (add instances)

After scaling:
├── CPU Usage is 48%
├── System: "Perfect! Keep it as is"
└── Action: No scaling needed

If CPU drops to 20%:
├── System: "Too low! Can reduce instances"
└── Action: Scale IN (remove instances)
```

**Predefined Metrics You Can Use**:
- `ASGAverageCPUUtilization` - Average CPU usage
- `ASGAverageNetworkIn` - Incoming network traffic
- `ASGAverageNetworkOut` - Outgoing network traffic
- `ALBRequestCountPerTarget` - Requests per instance (for load balancer)

**Example Configuration**:
```
Metric: CPU Utilization
Target Value: 70%
Scaling Behavior:
├── If CPU > 70%: Add instances
└── If CPU < 70%: Remove instances (gradually)
```

**Advantages**:
- Simple to set up
- AWS manages CloudWatch alarms automatically
- Scales smoothly following demand
- Less configuration needed

**Disadvantages**:
- Less fine-grained control
- Takes time to scale in (gradual, for stability)

**When to Use**:
- When you want simple, automatic scaling
- For applications with variable but predictable load
- When you're not sure about exact scaling thresholds

---

### 2. Step Scaling

**What it does**: Uses multiple thresholds with different scaling adjustments based on alarm breach size.

**How it works**: Different steps for different demand levels - like a car transmission with multiple gears.

```
Example: Scaling OUT based on CPU Utilization

CPU Range      Action
──────────────────────────────────
40% - 60%      No scaling
60% - 75%      Add 1 instance (mild demand)
75% - 85%      Add 3 instances (moderate demand)
85%+           Add 5 instances (high demand)

Step Adjustments Table:
┌──────────────┬──────────────┬────────────────┐
│ Lower Bound  │ Upper Bound  │ Adjustment     │
├──────────────┼──────────────┼────────────────┤
│ 0%           │ 15%          │ 0 instances    │
│ 15%          │ 25%          │ +1 instance    │
│ 25%          │ +∞           │ +3 instances   │
└──────────────┴──────────────┴────────────────┘
```

**Configuration Example**:
```
Create CloudWatch Alarm:
├── Metric: CPU Utilization
├── Threshold: > 60%
└── Evaluation: 2 consecutive minutes

Create Step Scaling Policy:
├── Alarm: CPU > 60%
├── Step 1: If CPU 60-75% → Add 10% capacity
├── Step 2: If CPU 75-85% → Add 20% capacity
└── Step 3: If CPU > 85% → Add 30% capacity
```

**Advantages**:
- Fine-grained control with multiple thresholds
- Better for sudden traffic spikes
- Proportional response to demand levels

**Disadvantages**:
- More complex configuration
- Manual CloudWatch alarm management
- More steps to create (alarms + policies)

**When to Use**:
- When you need precise control
- For applications with predictable traffic patterns
- When different demand levels need different responses

---

### 3. Simple Scaling

**What it does**: Triggers a single scaling adjustment when a metric exceeds a threshold.

**How it works**: One threshold = one action. If CPU > 80%, add 2 instances. Simple!

```
Example: Scale OUT on High CPU

Threshold: CPU > 80%
Action: Add 2 instances
Cooldown: 5 minutes (wait before next scaling)

Timeline:
T=0min    → CPU jumps to 85%
T=0-2min  → Alarm evaluates (2 data points)
T=2min    → Alarm triggers, add 2 instances
T=2-7min  → COOLDOWN period (no more scaling)
T=7min    → Can scale again if needed
```

**Configuration Example**:
```
Scale Out Policy:
├── Metric: CPU Utilization
├── Threshold: > 80%
├── Action: Add 2 instances
└── Cooldown: 300 seconds (5 minutes)

Scale In Policy:
├── Metric: CPU Utilization
├── Threshold: < 30%
├── Action: Remove 1 instance
└── Cooldown: 300 seconds
```

**Advantages**:
- Easiest to understand and implement
- Good for basic scaling needs
- Fast response time

**Disadvantages**:
- No graduated response (same action always)
- Can cause "flapping" (scaling in and out repeatedly)
- Requires cooldown period to prevent rapid scaling

**When to Use**:
- For simple, straightforward scaling
- When you're just getting started
- For applications with stable, predictable demand

---

### Comparison: Target Tracking vs Step vs Simple

| Feature | Target Tracking | Step Scaling | Simple Scaling |
|---------|-----------------|--------------|----------------|
| **Ease of Setup** | Very Easy | Moderate | Easy |
| **Control Level** | Low | High | Very Low |
| **Responsiveness** | Good | Excellent | Good |
| **Manual Alarms** | No (AWS manages) | Yes | Yes |
| **Flapping Risk** | Low | Low | High |
| **Best For** | Most use cases | Spiky traffic | Beginners |
| **Adjustment Types** | Fixed to target | Percentage/Count | Fixed value |

---

## Scheduled Scaling

**What it does**: Scales your infrastructure at specific times, based on a schedule.

**Use Case**: You know traffic increases every Monday morning at 8 AM, so you pre-scale your infrastructure.

### How It Works:

```
Monday, 8:00 AM → Traffic expected to spike
7:50 AM         → Scheduled action increases desired capacity
8:00 AM         → Infrastructure ready for traffic
5:00 PM         → Traffic decreases, schedule decreases capacity
```

### Example Configuration:

```
Scheduled Action 1: "Monday Morning Scale Out"
├── Recurrence: 0 8 * * 1 (8 AM every Monday)
├── Desired Capacity: 10 instances
├── Min Capacity: 5
└── Max Capacity: 20

Scheduled Action 2: "Weekend Scale Down"
├── Recurrence: 0 19 * * 5 (7 PM every Friday)
├── Desired Capacity: 2 instances
├── Min Capacity: 1
└── Max Capacity: 5
```

### Cron Expression Format:
```
Minute   Hour   Day   Month   Day_of_Week
  0  -   23  -  1-31  -  1-12  -  0-6 (0=Sunday)

Examples:
8 0 * * * = 8:00 AM every day
0 8 * * 1 = 8:00 AM every Monday
0 18 * * 5 = 6:00 PM every Friday
0 0 1 * * = Midnight on 1st of every month
*/15 * * * * = Every 15 minutes
```

### Advantages:
- Predictable, planned scaling
- No metric monitoring needed
- Great for known traffic patterns
- Cost optimization for recurring events

### Disadvantages:
- Doesn't handle unexpected traffic spikes
- Requires knowing your traffic patterns
- Needs manual adjustment if patterns change

### When to Use:
- Business hours scaling (9 AM - 6 PM)
- Weekly patterns (more traffic mid-week)
- Seasonal events (holiday sales)
- Regular maintenance windows

---

## Predictive Scaling

**What it does**: Uses machine learning to predict future demand and scale BEFORE traffic arrives.

**Why it's cool**: Instead of reacting to high CPU (too late!), it predicts the surge and scales proactively.

### How Machine Learning Works:

```
Step 1: Learning Phase (First 14 days)
├── Analyze historical usage patterns
├── Identify recurring spikes and trends
└── Build prediction model

Step 2: Prediction
├── Forecasts demand for next 48 hours
├── Updates forecast every 6 hours
└── Accounts for weekly/seasonal patterns

Step 3: Proactive Scaling
├── Schedules capacity increases 1 hour before predicted surge
├── Has instances ready before traffic arrives
└── Prevents performance degradation
```

### Real-World Example:

```
Traffic Pattern:
├── Every Monday at 8 AM: Traffic spikes 400%
├── Every Friday at 5 PM: Traffic drops 80%
└── Holidays: Special patterns

Predictive Scaling:
├── Learns this pattern in first 2 weeks
├── Monday 7:00 AM: Automatically scales to 12 instances
├── Friday 4:00 PM: Automatically scales down to 2 instances
├── Holiday: Uses historical holiday data to predict
└── Result: Zero performance issues, lower costs
```

### Two Modes:

**1. Forecast Only (Testing Mode)**
- Generates predictions without taking action
- Let you verify accuracy before enabling
- Useful for tuning and validation

```
Predicted Capacity: 15 instances
Actual Capacity: 8 instances
(No change yet - just showing forecast)
```

**2. Forecast and Scale (Active Mode)**
- Automatically adjusts capacity based on forecasts
- Scales OUT when prediction shows increase
- Requires dynamic scaling for scale-in

```
Predicted Capacity: 15 instances
Actual Capacity: 8 instances
Action Taken: Scale to 15 instances
```

### Configuration:

```
Predictive Scaling Policy:
├── Metric Type: ASGAverageCPUUtilization
├── Target Value: 50%
├── Mode: Forecast and Scale
├── Buffer: 10% (scale 10% higher than predicted)
├── Warmup Period: 300 seconds
└── Max Capacity Behavior: Allow exceeding max if needed
```

### Advantages:
- Proactive vs reactive
- Eliminates performance dips
- Handles recurring patterns automatically
- Works with dynamic scaling for complete solution

### Disadvantages:
- Needs 14 days of historical data
- Doesn't scale-in (use dynamic scaling for that)
- Less useful for unpredictable workloads
- Initial tuning period required

### When to Use:
- Applications with recurring traffic patterns
- Business hours traffic (9-5)
- Weekly spikes (mid-week busy)
- Seasonal patterns (holidays, back-to-school)
- Don't use: For new apps without history, unpredictable spikes

---

## Combining Scaling Policies (Best Practice)

Don't use just one policy - combine them!

### Recommended Strategy:

```
Target Tracking Scaling (Primary)
└── Maintains 70% CPU at all times

        ↓

Predictive Scaling (Proactive)
└── Anticipates spikes based on history

        ↓

Scheduled Scaling (Planned Events)
└── Handles known traffic patterns
```

### Example: E-commerce Website

```
1. Target Tracking (Always Active)
   ├── Metric: ALBRequestCountPerTarget
   ├── Target: 1000 requests/minute per instance
   └── Ensures consistent response time

2. Predictive Scaling (Enabled)
   ├── Learns from 2 weeks of traffic
   ├── Predicts lunch hour spike at 12 PM
   └── Scales up at 11:45 AM proactively

3. Scheduled Scaling (For Known Events)
   ├── Black Friday: Scale to 50 instances at 6 AM
   ├── Christmas: Increased capacity all month
   └── Regular: Weekend scale-down at 5 PM

Result:
✅ Handles unexpected spikes (Target Tracking)
✅ Ready before predictable spikes (Predictive)
✅ Optimized for special events (Scheduled)
✅ Always cost-efficient
```

---

## Best Practices

### 1. Choose the Right Metrics

```
Good Metrics (proportional to capacity):
✅ CPU Utilization
✅ Network I/O
✅ Request Count Per Target (ALB)
✅ Custom application metrics

Bad Metrics (don't scale proportionally):
❌ Total request count (doesn't change with capacity)
❌ Latency (doesn't scale linearly)
❌ Queue depth (without normalization)
```

### 2. Set Appropriate Target Values

```
General Guidelines:
├── CPU: 60-80% (leave room for spikes)
├── Network: 70-80% (depends on type)
├── Request Count: Depends on app
└── Memory: 75-85% (reserve for buffer)

Example:
├── Too Low (50%): Over-provisioning, wasted money
├── Too High (95%): Risk of performance issues
└── Just Right (70%): Balance of performance and cost
```

### 3. Set Min/Max Capacity Wisely

```
Min Capacity:
├── Should handle baseline traffic
├── Example: 2 instances minimum
└── Ensures availability during scale-in

Max Capacity:
├── Budget constraint
├── Performance requirement
├── Example: 20 instances maximum
└── Prevents runaway costs
```

### 4. Configure Warmup Periods

```
Warmup Period: Time before new instance counts toward metrics

Too Short (30 seconds):
├── Instance not fully ready
├── Skewed metrics
└── Unstable scaling

Too Long (10 minutes):
├── Slow response to additional spikes
├── Unnecessary idle instances
└── Higher costs

Recommended:
├── 300 seconds (5 minutes) for most apps
├── 60 seconds for highly optimized apps
└── Test and adjust based on startup time
```

### 5. Prevent Flapping (Oscillation)

Flapping = Continuous scale-in/scale-out cycle

```
Example of Flapping:
T=0:00 → CPU 82%, scale OUT to 5 instances
T=0:03 → CPU drops to 58%, scale IN to 3 instances
T=0:06 → CPU jumps to 80%, scale OUT to 5 instances
T=0:09 → CPU drops again... infinite loop!

Solution:
├── Use target tracking (handles this automatically)
├── Set margin between scale-out and scale-in thresholds
│   Example: Scale out at 75%, scale in at 40%
├── Use longer cooldown periods
└── Combine multiple policies
```

### 6. Enable Detailed Monitoring

```
CloudWatch Metrics:
├── Standard: 5-minute intervals
├── Detailed: 1-minute intervals

Recommendation:
├── Enable detailed monitoring for auto scaling
├── Allows faster response to changes
├── Minimal additional cost
```

### 7. Use Multiple Policies Together

```
Multiple Target Tracking Policies:
├── Policy 1: Target CPU at 70%
├── Policy 2: Target ALB Request Count at 1000/min
└── Logic: Scale OUT if ANY policy wants more
           Scale IN only if ALL policies want less

Multiple Scaling Types:
├── Dynamic (reactive to current load)
├── Scheduled (proactive for known times)
└── Predictive (AI-based forecasting)
```

---

## Real-World Examples

### Example 1: SaaS Application (Standard Business Hours)

**Scenario**: Web application used 9 AM - 6 PM, mostly idle outside

```
ASG Configuration:
├── Min: 1 instance
├── Desired: 2 instances
├── Max: 10 instances

Scaling Policies:

1. Target Tracking (Primary)
   ├── Metric: CPU Utilization
   ├── Target: 75%
   └── Always active

2. Scheduled Scaling (Cost Optimization)
   ├── 8:00 AM Mon-Fri: Set desired = 3
   ├── 6:00 PM Mon-Fri: Set desired = 1
   ├── Weekends: Set desired = 1
   └── Purpose: Have instances ready before business hours

Result:
├── 9 AM: 3 instances ready
├── Noon: Auto scales to 5-7 based on usage
├── 6 PM: Scales back to 1
└── Cost: 40% lower than always running
```

### Example 2: E-Commerce Site (Variable Demand)

**Scenario**: Traffic varies by day, special events

```
ASG Configuration:
├── Min: 3 instances
├── Desired: 5 instances
├── Max: 30 instances

Scaling Policies:

1. Target Tracking (Always On)
   ├── Metric: ALBRequestCountPerTarget
   ├── Target: 1500 requests/minute
   └── Handles normal fluctuations

2. Scheduled Scaling (Predictable Patterns)
   ├── Monday-Thursday 10 AM: min=5, desired=8
   ├── Friday 10 AM: min=10, desired=15 (busy day)
   ├── Weekend 10 AM: min=3, desired=4 (slower)
   └── Every night 2 AM: desired=3 (maintenance window)

3. Predictive Scaling (Historical Patterns)
   ├── Learns lunch hour surge (12-1 PM)
   ├── Learns evening surge (6-9 PM)
   ├── Learns weekend slowdown
   └── Scales proactively 30 min before spikes

Result:
├── No more manual scaling
├── All traffic spikes handled automatically
├── Predictive prevents performance dips
└── Estimated cost: 35% savings vs dedicated capacity
```

### Example 3: Real-Time Analytics (Batch Jobs)

**Scenario**: Heavy computation jobs, variable frequency

```
ASG Configuration:
├── Min: 1 instance
├── Desired: 2 instances
├── Max: 50 instances

Scaling Policies:

1. Step Scaling (For Gradual Load)
   ├── Monitor: Custom metric (queue depth per instance)
   ├── 100-150 jobs: Add 2 instances
   ├── 150-300 jobs: Add 5 instances
   ├── 300+ jobs: Add 10 instances
   └── Handles burst loads incrementally

2. Cooldown: 120 seconds
   └── Prevents overscaling during processing

3. Instance Warmup: 600 seconds
   └── Time for instance to start processing jobs

Scaling Out (High Load):
├── Queue depth = 500 jobs
├── Calculate: 500/50 per instance = 10 instances needed
├── Action: Scale to 15 instances (safety margin)
└── Jobs: Start processing immediately

Scaling In (Low Load):
├── Queue depth = 20 jobs
├── No more job submissions
├── Action: Gradually reduce (every 2 minutes)
└── Cost: $50/hour when idle, $2000/hour when running

Result:
├── Handles variable workload efficiently
├── No job queuing delays
└── Cost-efficient for bursty pattern
```

---

## Troubleshooting Common Issues

### Issue: Not Scaling Out When Expected

**Possible Causes**:
1. Metric is missing data points
   - Solution: Check CloudWatch metrics, enable detailed monitoring

2. Max capacity reached
   - Solution: Increase max capacity limit

3. Warmup period still active
   - Solution: Wait for instances to warm up

4. Cooldown period active (Simple Scaling)
   - Solution: Wait or reduce cooldown period

### Issue: Continuous Flapping (Scale In/Out Loop)

**Solution**:
```
1. Use Target Tracking instead of Step/Simple
2. Increase gap between scale-in and scale-out thresholds
3. Increase cooldown or warmup periods
4. Increase target value (less aggressive scaling)
5. Combine with Scheduled Scaling for stability
```

### Issue: High Costs

**Solutions**:
1. Lower target metric values (scale sooner)
2. Lower max capacity limit
3. Add Scheduled Scaling to scale down during off-hours
4. Use Reserved Instances for baseline load
5. Use Spot Instances for variable load

### Issue: Poor Performance

**Solutions**:
1. Increase target metric value
2. Lower warmup period
3. Enable Predictive Scaling
4. Use Scheduled Scaling to pre-scale
5. Check if metric choice is appropriate

---

## Summary Table: When to Use Each Policy

| Use Case | Best Policy | Why |
|----------|-------------|-----|
| New application, simple needs | Target Tracking | Easiest, AWS manages alarms |
| Spiky traffic, need control | Step Scaling | Graduated response |
| Unknown patterns | Target Tracking | Adapts automatically |
| Predictable daily patterns | Scheduled Scaling | Cost optimization |
| Recurring traffic spikes | Predictive Scaling | Proactive scaling |
| Business hours only | Scheduled + Target | Planned + reactive |
| E-commerce site | All three combined | Best of all worlds |
| IoT sensors, real-time | Step or Target | Sensitive to load |
| Batch processing | Step Scaling | Handle bursts |
| Microservices | Target Tracking | Simple per-service |

---

## Key Takeaways

✅ **Scaling policies automate infrastructure management**
- No more manual instance management
- Consistent performance
- Lower costs

✅ **Choose the right policy for your use case**
- Target Tracking: Most situations
- Step Scaling: Precise control
- Simple Scaling: Beginners
- Scheduled: Known patterns
- Predictive: Recurring spikes

✅ **Combine policies for best results**
- Target Tracking (reactive)
- Scheduled Scaling (planned)
- Predictive Scaling (proactive)

✅ **Monitor and tune regularly**
- Review metrics quarterly
- Adjust thresholds based on patterns
- Test forecast accuracy
- Optimize cost vs performance

✅ **Prevent flapping and performance issues**
- Set appropriate thresholds
- Use warmup periods
- Enable detailed monitoring
- Test in production gradually

---

## Additional Resources

### AWS Documentation
- EC2 Auto Scaling: https://docs.aws.amazon.com/autoscaling/
- Target Tracking: https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-target-tracking.html
- Step Scaling: https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-simple-step.html
- Scheduled Scaling: https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-scheduled-scaling.html
- Predictive Scaling: https://docs.aws.amazon.com/autoscaling/ec2/userguide/predictive-scaling-policy-overview.html

### Next Steps
1. Create your first Auto Scaling Group
2. Set up Target Tracking policy
3. Monitor for 1-2 weeks
4. Add Scheduled Scaling if needed
5. Enable Predictive Scaling after 14 days

---

**Last Updated**: January 2026  
**Author**: AWS DevOps Learning Guide  
**For**: Entry-level DevOps/Cloud Engineers
