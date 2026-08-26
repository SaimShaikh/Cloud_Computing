# Amazon EC2 Flex Instances --- End-to-End Guide

> **Status:** Current as of August 26, 2026\
> **Scope:** What Flex instances are, why AWS introduced them, how they
> work internally, billing, performance behavior, Flex vs Burstable vs
> Fixed, families, sizing, deployment, monitoring, troubleshooting,
> architecture decisions, and practical examples.

------------------------------------------------------------------------

## 1. Executive Summary

Amazon EC2 **Flex instances** are lower-priced variants of selected
modern EC2 instance families. They are designed for workloads that need
the resources of a normal EC2 instance but **do not continuously use
100% of the available CPU**.

The key idea is:

-   A Flex instance provides a **reliable 40% CPU baseline**.
-   When the workload needs more CPU, it can **burst above 40% and reach
    up to 100% CPU performance**.
-   AWS states that Flex instances can provide up to 100% CPU
    performance for **95% of the time over a 24-hour window**.
-   Flex instances are **not T-family burstable instances** and do **not
    use the normal CPU-credit model** used by T2/T3/T4g.
-   Flex is primarily a **cost/performance optimization**, not a
    replacement for a fixed-performance instance when sustained high CPU
    is required.

AWS currently lists these Flex families:

  Family     Category            Typical purpose
  ---------- ------------------- ---------------------------------
  C7i-flex   Compute optimized   CPU-heavy general workloads
  C8i-flex   Compute optimized   Newer CPU-intensive workloads
  M7i-flex   General purpose     Balanced CPU + memory workloads
  M8i-flex   General purpose     Newer balanced workloads
  R8i-flex   Memory optimized    Memory-heavy workloads

The common Flex size range is generally **large through 16xlarge**, with
up to **64 vCPUs** and, depending on family, up to **512 GiB memory**.

**The one-line mental model:**

> **Fixed instance = full CPU whenever required**\
> **T instance = credit-based burst model**\
> **Flex instance = lower-cost instance with a reliable 40% CPU baseline
> and time-based burst capability**

------------------------------------------------------------------------

# 2. Why Did AWS Introduce Flex Instances?

A common EC2 sizing problem is **over-provisioning**.

Suppose an application needs:

-   32 GiB RAM
-   8 vCPUs
-   100% CPU for short periods
-   but normally uses only 15--30% CPU

A traditional fixed-performance instance gives you the full CPU
capability all the time. That is excellent for performance, but you may
be paying for compute capacity that is mostly idle.

AWS Flex instances target this middle ground.

Instead of forcing customers to choose between:

1.  a fixed-performance instance that costs more, or
2.  a T-family burstable instance with CPU credits,

Flex gives a third model:

``` text
                    EC2 CPU behavior

             Fixed                 Flex                 Burstable
               │                    │                      │
               │                    │                      │
CPU            │  100% ─────────   │     burst to 100%   │
performance    │                    │       when needed   │
               │                    │                      │
               │                    │  40% baseline       │
               │                    │                      │
               │                    │                      │
               │                    │                      │
               └────────────────────┴──────────────────────┴── Time
```

The objective is to match the price to the **actual utilization
pattern** of common workloads.

------------------------------------------------------------------------

# 3. What Exactly Is a Flex Instance?

A Flex instance is an EC2 instance type designed around **variable CPU
demand**.

It has:

-   a fixed amount of vCPU
-   a fixed amount of memory
-   defined network capability
-   defined EBS bandwidth
-   a **40% CPU baseline**
-   the ability to exceed that baseline
-   no T-family-style CPU-credit accounting

The important point is:

> **40% is a baseline, not a hard CPU limit.**

If you launch a `m8i-flex.2xlarge`, AWS does not mean:

> "You can only use 40% CPU."

Instead, it means:

> "The instance has reliable CPU performance at the 40% baseline and can
> go higher when required, subject to the Flex performance model."

------------------------------------------------------------------------

# 4. The 40% Baseline --- What Does It Mean?

This is one of the most important concepts.

AWS states that Flex instances provide a reliable CPU baseline of
**40%**.

Think of the instance's total CPU capacity as:

``` text
0% ───────────── 40% ─────────────────────────── 100%
       baseline                 burst range
```

At or below approximately 40%:

-   the workload is operating within the normal baseline
-   there is no CPU-credit balance to manage
-   the instance is designed to provide that baseline reliably

Above 40%:

-   the instance can use additional CPU capacity
-   it can scale toward full CPU performance
-   AWS's Flex model governs how long maximum burst performance can be
    maintained

### Important clarification

The 40% figure is about **CPU performance/utilization**, not memory.

For example:

``` text
m8i-flex.2xlarge

vCPU = 8
RAM  = 32 GiB
```

The baseline is not:

``` text
8 vCPU × 40% = "only 3.2 vCPU are available"
```

It is better to think of it as **40% aggregate CPU performance across
the instance**.

CPU usage is workload-dependent. One process could heavily use one vCPU
while other vCPUs are idle, and the aggregate CloudWatch CPU percentage
could still remain below 40%.

------------------------------------------------------------------------

# 5. How the Flex Burst Model Works

AWS describes Flex as providing up to **100% CPU performance for 95% of
the time over a 24-hour window**.

This is fundamentally different from CPU credits.

Conceptually:

``` text
24-hour observation window

|----------------------------------------------------------|
|                                                          |
|   95% of time: can provide up to full CPU performance   |
|                                                          |
|----------------------------------------------------------|
|  remaining portion may experience reduced burst         |
|  throughput depending on sustained CPU demand           |
|----------------------------------------------------------|
```

### Very important

Do not interpret this as:

> "Every 24 hours I get a bucket of 95% CPU credits."

That is **not** how Flex works.

There is no normal T-family credit balance that you watch going:

``` text
100 credits
90 credits
80 credits
...
0 credits
```

Flex uses a different performance model.

------------------------------------------------------------------------

# 6. What Happens When CPU Is Continuously High?

Suppose a Flex instance runs at:

``` text
10% CPU → 20% → 30% → 40%
```

This is comfortably within the baseline.

Now suppose it stays at:

``` text
80% → 90% → 100%
```

for a long period.

The instance can burst, but sustained high CPU usage is exactly the
scenario where Flex becomes less attractive.

AWS explicitly notes that Flex instances running at high CPU utilization
consistently above the baseline for long periods can experience a
**gradual reduction in maximum burst CPU throughput**.

Therefore:

> Flex is intended for workloads with **variable CPU demand**, not
> workloads that need maximum CPU continuously.

If the workload requires sustained high CPU, use the comparable
**non-Flex fixed-performance instance**.

------------------------------------------------------------------------

# 7. Flex vs Fixed Performance

A fixed-performance instance provides full CPU resources whenever the
workload needs them.

Example:

``` text
Application CPU demand:

100% ────────████████████████████████████
 80% ────────████████████████████████████
 60% ────────████████████████████████████
 40% ────────████████████████████████████
 20% ────────████████████████████████████
              time →

Fixed:
Full CPU capability is continuously available.
```

Flex:

``` text
100% ────────────────╮    ╭──────────────
                     │    │
 80% ────────────────│────│──────────────
                     │    │
 40% ════════════════╪════╪══════════════
                     baseline
```

### Choose Fixed when:

-   CPU is continuously high
-   CPU latency is critical
-   workload needs predictable sustained maximum throughput
-   large instance sizes are required
-   continuous high network/EBS performance is required
-   HPC or large-scale compute is involved
-   video encoding/streaming requires sustained CPU
-   large database workloads continuously consume CPU

------------------------------------------------------------------------

# 8. Flex vs Burstable T Instances

This is the comparison that causes the most confusion.

## 8.1 T-family model

T2/T3/T4g instances use **CPU credits**.

When CPU utilization is below baseline:

``` text
CPU credits are earned
```

When CPU utilization is above baseline:

``` text
CPU credits are spent
```

In Standard mode, if credits are exhausted:

``` text
burst performance eventually falls toward baseline
```

In Unlimited mode:

``` text
instance can continue bursting
but surplus CPU usage can create additional charges
```

The credit system is documented by AWS as the defining mechanism for
burstable performance instances.

## 8.2 Flex model

Flex does not use that T-family CPU-credit model.

Instead:

``` text
Flex
 │
 ├── 40% reliable baseline
 │
 ├── can burst above baseline
 │
 ├── can reach up to 100% CPU performance
 │
 ├── 95% of time over a 24-hour window
 │
 └── no T-family CPU-credit accounting
```

### Core difference

  -----------------------------------------------------------------------
  Feature                 Flex                    T3/T4g Burstable
  ----------------------- ----------------------- -----------------------
  CPU baseline            40%                     Family/size dependent

  CPU credits             No normal T-family      Yes
                          credit model            

  Unlimited mode          Not applicable as       Yes
                          T-family CPU-credit     
                          model                   

  Additional burst CPU    No T-family credit      Possible in Unlimited
  credit charge           charge                  mode

  Burst capability        Yes                     Yes

  Maximum burst           Up to 100% CPU          Up to full CPU while
                          performance             credits/Unlimited
                                                  behavior allows

  Main design goal        Lower cost for          Very economical
                          under-utilized modern   variable workloads
                          instances               

  Typical sizes           Common sizes through    Small through large and
                          16xlarge                beyond depending on
                                                  family

  Best fit                Modern production       Small/medium variable
                          workloads needing       workloads
                          larger resources        
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 9. Flex vs Burstable --- Simple Real-World Example

Suppose you have a web server.

CPU behavior:

``` text
Normal traffic:
20–30%

Traffic spike:
80–100%

After spike:
20–30%
```

Both Flex and T-family instances can be candidates.

But the mechanisms are different.

### T3

``` text
Low CPU
   ↓
earn credits

Traffic spike
   ↓
spend credits

Credits exhausted
   ↓
Standard mode → baseline limitation
Unlimited mode → continue and potentially pay extra
```

### Flex

``` text
Low/normal CPU
   ↓
operate around 40% baseline or below

Traffic spike
   ↓
burst above baseline

Sustained high CPU
   ↓
maximum burst capability can gradually reduce
```

The decision should therefore be based on **workload size, performance
requirements, CPU pattern, and cost**, not simply the word "burst."

------------------------------------------------------------------------

# 10. Flex Instance Families

## 10.1 C7i-flex

**Category:** Compute optimized

Powered by 4th Generation Intel Xeon Scalable processors.

Typical workloads:

-   web servers
-   application servers
-   databases
-   caches
-   Apache Kafka
-   Elasticsearch
-   compute-intensive services

Current common sizes include:

  Size                  vCPU    Memory
  ------------------- ------ ---------
  c7i-flex.large           2     4 GiB
  c7i-flex.xlarge          4     8 GiB
  c7i-flex.2xlarge         8    16 GiB
  c7i-flex.4xlarge        16    32 GiB
  c7i-flex.8xlarge        32    64 GiB
  c7i-flex.12xlarge       48    96 GiB
  c7i-flex.16xlarge       64   128 GiB

AWS documents 40% baseline CPU performance for the family.

------------------------------------------------------------------------

## 10.2 C8i-flex

**Category:** Compute optimized

Powered by Intel Xeon 6 processors.

Use cases include:

-   compute-intensive web servers
-   application servers
-   caching
-   Kafka
-   Elasticsearch
-   batch processing
-   distributed analytics
-   HPC-type workloads where sustained maximum CPU is not required
-   CPU-based inference

Current common sizes:

  Size                  vCPU    Memory
  ------------------- ------ ---------
  c8i-flex.large           2     4 GiB
  c8i-flex.xlarge          4     8 GiB
  c8i-flex.2xlarge         8    16 GiB
  c8i-flex.4xlarge        16    32 GiB
  c8i-flex.8xlarge        32    64 GiB
  c8i-flex.12xlarge       48    96 GiB
  c8i-flex.16xlarge       64   128 GiB

------------------------------------------------------------------------

## 10.3 M7i-flex

**Category:** General purpose

Best when you need a balance of:

-   CPU
-   RAM
-   networking
-   general application performance

Typical workloads:

-   web applications
-   application servers
-   microservices
-   databases
-   virtual desktops
-   batch processing
-   enterprise applications

------------------------------------------------------------------------

## 10.4 M8i-flex

**Category:** General purpose

Powered by Intel Xeon 6 processors.

Typical workloads:

-   web servers
-   application servers
-   microservices
-   data stores
-   virtual desktops
-   enterprise applications
-   general databases

Current common sizes:

  Size                  vCPU    Memory
  ------------------- ------ ---------
  m8i-flex.large           2     8 GiB
  m8i-flex.xlarge          4    16 GiB
  m8i-flex.2xlarge         8    32 GiB
  m8i-flex.4xlarge        16    64 GiB
  m8i-flex.8xlarge        32   128 GiB
  m8i-flex.12xlarge       48   192 GiB
  m8i-flex.16xlarge       64   256 GiB

------------------------------------------------------------------------

## 10.5 R8i-flex

**Category:** Memory optimized

This is designed for workloads where memory is more important than raw
CPU.

Typical workloads:

-   memory-heavy databases
-   Redis/Memcached
-   in-memory analytics
-   enterprise applications
-   real-time analytics
-   memory-intensive application servers

Current common sizes:

  Size                  vCPU    Memory
  ------------------- ------ ---------
  r8i-flex.large           2    16 GiB
  r8i-flex.xlarge          4    32 GiB
  r8i-flex.2xlarge         8    64 GiB
  r8i-flex.4xlarge        16   128 GiB
  r8i-flex.8xlarge        32   256 GiB
  r8i-flex.12xlarge       48   384 GiB
  r8i-flex.16xlarge       64   512 GiB

**Important:** Always verify the current AWS regional availability and
exact size catalog before deployment because instance availability
changes by Region.

------------------------------------------------------------------------

# 11. How to Read a Flex Instance Name

Example:

``` text
m8i-flex.4xlarge
```

Breakdown:

``` text
m      = general purpose family
8      = generation
i      = Intel
flex   = Flex performance model
4xlarge = instance size
```

Another example:

``` text
r8i-flex.8xlarge
```

means:

``` text
r = memory optimized
8 = generation
i = Intel
flex = Flex
8xlarge = size
```

------------------------------------------------------------------------

# 12. Flex Is Not a Separate EC2 Service

This is an important architectural point.

Flex is **not** something like:

``` text
EC2
 ├── Standard EC2
 ├── Flex service
 └── T service
```

Instead:

``` text
Amazon EC2
   │
   └── Instance types
        │
        ├── Fixed-performance families
        ├── Burstable T families
        └── Flex variants
```

Flex is a **performance characteristic of specific EC2 instance types**.

------------------------------------------------------------------------

# 13. How to Launch a Flex Instance

You launch a Flex instance just like a normal EC2 instance.

## Console flow

``` text
AWS Console
   ↓
EC2
   ↓
Instances
   ↓
Launch instance
   ↓
Choose AMI
   ↓
Choose instance type
   ↓
Select:
m8i-flex.large
m8i-flex.xlarge
c8i-flex.large
...
   ↓
Configure networking
   ↓
Configure storage
   ↓
Configure IAM
   ↓
Security Group
   ↓
Launch
```

There is no separate "Enable Flex" switch.

The **instance type name itself** determines that you are launching a
Flex instance.

------------------------------------------------------------------------

# 14. AWS CLI Example

Example:

``` bash
aws ec2 run-instances \
  --image-id ami-xxxxxxxxxxxxxxxxx \
  --instance-type m8i-flex.large \
  --subnet-id subnet-xxxxxxxx \
  --security-group-ids sg-xxxxxxxx \
  --iam-instance-profile Name=MyEC2Role \
  --key-name my-key
```

The important parameter is:

``` bash
--instance-type m8i-flex.large
```

------------------------------------------------------------------------

# 15. Terraform Example

``` hcl
resource "aws_instance" "flex" {
  ami           = "ami-xxxxxxxxxxxxxxxxx"
  instance_type = "m8i-flex.large"

  subnet_id = aws_subnet.private.id

  vpc_security_group_ids = [
    aws_security_group.app.id
  ]

  iam_instance_profile = aws_iam_instance_profile.ec2.name

  tags = {
    Name = "flex-app-server"
  }
}
```

The infrastructure workflow is otherwise normal EC2.

------------------------------------------------------------------------

# 16. What Actually Gets Billed?

This is where people often mix up **instance performance** with
**billing**.

A Flex instance is still billed as EC2 compute capacity.

The major cost variables are:

``` text
Total EC2 workload cost
=
EC2 instance compute
+
EBS
+
EBS snapshots
+
Data transfer
+
Public IPv4
+
Operating-system/license charges
+
Other attached AWS services
```

Flex itself does not mean:

> "AWS bills every CPU percentage separately."

That is the wrong mental model.

------------------------------------------------------------------------

# 17. Flex Compute Billing

For On-Demand EC2:

``` text
Compute cost
=
instance hourly/secondly rate
× running duration
```

AWS states that On-Demand EC2 instances are billed by the second with a
**60-second minimum**.

For example, conceptually:

``` text
Price = $X/hour

Running:
1 hour
→ $X

Running:
10 hours
→ 10 × $X
```

Actual prices depend on:

-   Region
-   instance type
-   operating system
-   tenancy
-   purchasing option
-   applicable pricing program

------------------------------------------------------------------------

# 18. Does Flex Charge Extra for CPU Bursting?

This is a major difference from T-family Unlimited mode.

Flex does **not** use the T-family CPU-credit billing model.

You should not expect a separate:

``` text
CPU credit balance
CPU surplus charge
CPU credit purchase
```

just because a Flex instance goes above 40% CPU.

The instance is purchased at its applicable EC2 rate.

However, **other AWS costs can still increase** because the workload
may:

-   process more data
-   transfer more data
-   use more EBS I/O
-   generate more logs
-   trigger more application-level services

So:

> **High CPU does not create a T-family CPU-credit surcharge, but the
> workload can still create other AWS charges.**

------------------------------------------------------------------------

# 19. Important Billing Variables

## 19.1 Region

EC2 pricing differs by Region.

For example:

``` text
us-east-1
ap-south-1
eu-west-1
```

can have different prices.

Always calculate cost in the Region where you actually deploy.

------------------------------------------------------------------------

## 19.2 Operating System

The AMI/OS can change the effective compute price.

Examples:

-   Linux
-   Windows
-   RHEL
-   SUSE

Windows and commercial software licensing can significantly change the
final cost.

------------------------------------------------------------------------

## 19.3 On-Demand

You pay for running usage without a long-term commitment.

Good for:

-   testing
-   uncertain workloads
-   short-lived environments
-   workloads where flexibility matters

------------------------------------------------------------------------

## 19.4 Savings Plans

Savings Plans can reduce compute cost in exchange for a usage
commitment.

This can be attractive for:

-   production environments
-   predictable workloads
-   long-running EC2 fleets

Always evaluate the commitment against actual usage.

------------------------------------------------------------------------

## 19.5 Spot

If the workload can tolerate interruption, Spot can provide major
discounts.

Good candidates:

-   stateless workers
-   batch jobs
-   CI runners
-   fault-tolerant distributed processing
-   scalable worker fleets

Do not choose Spot merely because Flex is inexpensive. They solve
different problems.

------------------------------------------------------------------------

## 19.6 EBS

Flex instances normally use EBS rather than local instance storage.

Your bill can include:

-   EBS volume capacity
-   provisioned IOPS where applicable
-   provisioned throughput where applicable
-   snapshots
-   data transfer associated with other services where applicable

So:

``` text
EC2 price ≠ complete server bill
```

------------------------------------------------------------------------

## 19.7 Public IPv4

A public IPv4 address can incur an AWS charge.

Do not forget this when calculating the total cost of a small EC2
deployment.

------------------------------------------------------------------------

## 19.8 Data Transfer

Traffic can generate additional costs depending on:

-   Internet traffic
-   cross-AZ traffic
-   cross-Region traffic
-   NAT Gateway
-   other AWS service paths

The network bandwidth shown in an instance specification is
**capacity**, not a monthly data-transfer allowance.

------------------------------------------------------------------------

# 20. A Practical Monthly Cost Formula

For a simple Linux On-Demand deployment:

``` text
Monthly EC2 compute
≈
On-Demand hourly rate
×
hours running
```

Then:

``` text
Total monthly AWS cost
≈

EC2 compute
+ EBS
+ snapshots
+ public IPv4
+ data transfer
+ NAT Gateway
+ CloudWatch/logging
+ load balancer
+ other services
```

For a real production environment, always calculate the whole
architecture instead of looking only at the EC2 line item.

------------------------------------------------------------------------

# 21. Flex and Auto Scaling

Flex works normally with EC2 Auto Scaling.

Example:

``` text
                    Application Load Balancer
                              │
                              ▼
                    Auto Scaling Group
                   ┌──────────┼──────────┐
                   ▼          ▼          ▼
              m8i-flex    m8i-flex    m8i-flex
```

This is actually a very good architecture for Flex.

Why?

Because Auto Scaling handles **capacity scaling**, while Flex handles
**CPU efficiency inside each instance**.

You can combine:

``` text
Flex
+
Auto Scaling
+
CloudWatch
+
ALB
```

to handle variable demand.

------------------------------------------------------------------------

# 22. Flex and Kubernetes

Flex instances can also be used as worker nodes for Kubernetes/EKS when
the instance type is supported in the relevant Region and configuration.

Example:

``` text
EKS
 │
 └── Managed Node Group
        │
        ├── m8i-flex.large
        ├── m8i-flex.large
        └── m8i-flex.large
```

The important question is not:

> "Does Kubernetes support Flex?"

The better question is:

> "Does my workload have a CPU profile suitable for Flex?"

For Kubernetes, consider:

-   CPU requests
-   CPU limits
-   pod density
-   memory pressure
-   daemonset overhead
-   cluster autoscaling
-   node utilization
-   workload latency

------------------------------------------------------------------------

# 23. Flex and Databases

Flex can be useful for databases when CPU demand is variable.

Potential examples:

-   development databases
-   moderate application databases
-   databases with bursty request patterns
-   workloads where memory is the dominant resource and CPU is not
    continuously saturated

But be careful with:

-   large production databases
-   sustained query-heavy workloads
-   CPU-bound analytics
-   high transaction workloads
-   workloads requiring consistent maximum CPU

For these cases, compare against the non-Flex equivalent.

------------------------------------------------------------------------

# 24. Flex and Web Servers

Web servers are one of the strongest candidates.

Example:

``` text
Night:
CPU = 10–20%

Normal business hours:
CPU = 25–40%

Traffic spike:
CPU = 70–100%

After spike:
CPU = 20–30%
```

This is exactly the kind of utilization pattern Flex is intended to
target.

Combine it with:

``` text
ALB
+
Auto Scaling
+
Flex instances
+
CloudWatch
```

for a strong cost/performance architecture.

------------------------------------------------------------------------

# 25. Flex and DevOps Workloads

Flex can be particularly useful for:

-   Jenkins servers
-   GitLab runners
-   CI/CD application servers
-   monitoring servers
-   internal tools
-   API servers
-   staging environments
-   development environments
-   automation servers
-   bastion-like workloads where supported
-   moderate build workloads

However, CI workers that are continuously CPU-bound may be better on
fixed-performance or ephemeral Spot capacity.

------------------------------------------------------------------------

# 26. Flex and AI/ML

Flex is **not automatically an AI instance**.

It is a CPU instance.

For:

-   CPU-based inference
-   preprocessing
-   orchestration
-   API layers
-   feature processing

Flex may be useful.

For:

-   GPU training
-   large GPU inference
-   high-end HPC
-   sustained CPU-heavy ML workloads

you should evaluate specialized EC2 families instead.

Do not select Flex simply because the workload is called "AI."

------------------------------------------------------------------------

# 27. Flex and Network Performance

Flex is primarily about **CPU performance/cost efficiency**.

Do not assume:

``` text
Flex = unlimited network
```

Network bandwidth is still instance-size and family dependent.

For example, a smaller Flex instance may have lower network bandwidth
than a larger one.

Therefore evaluate:

``` text
CPU
RAM
Network
EBS
```

independently.

------------------------------------------------------------------------

# 28. Flex and EBS Performance

The same principle applies to EBS.

An instance can have:

``` text
high CPU capability
```

but still have a specific:

``` text
EBS bandwidth / IOPS capability
```

If the workload is EBS-bound rather than CPU-bound, changing from a
fixed instance to Flex may not solve the problem.

Always identify the actual bottleneck.

------------------------------------------------------------------------

# 29. The Four Main EC2 CPU Models

It is useful to classify EC2 CPU behavior into four practical groups.

## 29.1 Fixed

``` text
Full CPU performance
+
Predictable sustained performance
```

Best for:

-   sustained CPU workloads
-   HPC
-   encoding
-   high-volume compute
-   large databases

------------------------------------------------------------------------

## 29.2 Burstable

``` text
Baseline
+
CPU credits
+
Burst
```

Best for:

-   small/medium workloads
-   variable CPU
-   development
-   low-cost applications

------------------------------------------------------------------------

## 29.3 Flex

``` text
40% baseline
+
burst capability
+
95%/24-hour performance model
+
no T-family CPU-credit accounting
```

Best for:

-   modern general workloads
-   under-utilized production servers
-   workloads needing larger RAM/vCPU sizes
-   cost optimization

------------------------------------------------------------------------

## 29.4 Specialized

Examples:

-   GPU
-   Inferentia
-   Trainium
-   HPC
-   storage optimized
-   high-memory
-   network optimized

Best when the bottleneck is not simply general CPU.

------------------------------------------------------------------------

# 30. Decision Tree --- Which One Should I Choose?

``` text
Start
 │
 ├── Is CPU continuously high?
 │       │
 │       ├── YES → Fixed-performance instance
 │       │
 │       └── NO
 │
 ├── Is the workload small and very cost sensitive?
 │       │
 │       ├── YES → Consider T3/T4g
 │       │
 │       └── NO
 │
 ├── Do you need a modern instance with more RAM/vCPU?
 │       │
 │       ├── YES → Consider Flex
 │       │
 │       └── NO
 │
 └── Benchmark Flex vs fixed vs burstable
```

------------------------------------------------------------------------

# 31. A Better Selection Method

Do not select based only on:

> "Flex is cheaper."

Instead measure:

``` text
1. Average CPU
2. CPU peak
3. Duration of peaks
4. Memory utilization
5. Network utilization
6. EBS utilization
7. Application latency
8. Error rate
9. Throughput
10. Cost
```

Then benchmark.

AWS itself recommends testing instance types with your own benchmark
workload.

------------------------------------------------------------------------

# 32. CloudWatch Metrics to Monitor

At minimum, monitor:

### EC2

-   `CPUUtilization`
-   `NetworkIn`
-   `NetworkOut`
-   `NetworkPacketsIn`
-   `NetworkPacketsOut`
-   `DiskReadOps`
-   `DiskWriteOps`
-   `DiskReadBytes`
-   `DiskWriteBytes`
-   EBS-related metrics

### Application

-   request latency
-   requests per second
-   error rate
-   queue depth
-   worker utilization
-   database latency

The most important Flex metric is usually:

``` text
CPUUtilization
```

but CPU alone is not enough.

------------------------------------------------------------------------

# 33. Monitoring Strategy for Flex

A useful dashboard:

``` text
┌──────────────────────────────────────────────┐
│ FLEX INSTANCE DASHBOARD                      │
├──────────────────────────────────────────────┤
│ CPU Utilization                              │
│ ───────────────────────────────               │
│ baseline reference: 40%                      │
│                                              │
│ Memory Utilization                           │
│ ───────────────────────────────               │
│                                              │
│ Network                                      │
│ ───────────────────────────────               │
│                                              │
│ EBS                                          │
│ ───────────────────────────────               │
│                                              │
│ Application Latency                          │
│ ───────────────────────────────               │
└──────────────────────────────────────────────┘
```

Create alarms for:

``` text
CPU > expected sustained threshold
+
Latency > SLO
+
Memory > threshold
+
EBS saturation
+
Network saturation
```

Do not create an alarm simply because CPU crossed 40%.

Crossing 40% is not automatically an incident.

------------------------------------------------------------------------

# 34. The 40% Baseline Is Not a CloudWatch Alarm Threshold

This is a common mistake.

Wrong:

``` text
CPU > 40%
→ alarm
→ Flex is failing
```

Correct:

``` text
CPU > 40%
→ instance is using burst capacity
→ observe duration and application behavior
```

A short spike to:

``` text
95%
```

may be perfectly healthy.

A workload sitting around:

``` text
90–100%
```

for hours is a very different situation.

------------------------------------------------------------------------

# 35. Capacity Planning

For a Flex workload, calculate:

``` text
Average CPU
Peak CPU
Peak duration
Memory requirement
Network requirement
EBS requirement
```

Example:

``` text
Average CPU = 25%
Peak CPU = 90%
Peak duration = 10 minutes
RAM = 20 GiB
Network = 2 Gbps
```

This profile is much more Flex-friendly than:

``` text
Average CPU = 75%
Peak CPU = 100%
Peak duration = 8 hours
RAM = 20 GiB
```

The second workload is much closer to a fixed-performance workload.

------------------------------------------------------------------------

# 36. Flex + Auto Scaling Is Often Better Than Oversizing

Instead of:

``` text
1 huge fixed instance
```

consider:

``` text
3 smaller Flex instances
+
Auto Scaling
+
Load Balancer
```

Example:

``` text
                 ALB
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
      Flex      Flex      Flex
       25%       30%       20%
```

During traffic spikes:

``` text
CPU increases
      ↓
Auto Scaling detects demand
      ↓
launch another Flex instance
      ↓
traffic distributed
      ↓
CPU falls
```

This can improve both resilience and cost efficiency.

------------------------------------------------------------------------

# 37. High Availability

Never confuse:

``` text
Flex
```

with:

``` text
High Availability
```

Flex is an instance performance/cost model.

High availability comes from architecture:

``` text
Multiple AZs
+
Auto Scaling
+
Load Balancer
+
stateless design
+
managed services
+
health checks
```

You can build a highly available Flex architecture, but Flex itself does
not provide HA.

------------------------------------------------------------------------

# 38. Flex and Instance Stop/Start

Flex follows normal EC2 lifecycle behavior.

You can:

-   start
-   stop
-   reboot
-   terminate
-   modify where the instance-type transition is supported

Stopping an instance stops instance compute billing, but attached
resources such as EBS can continue to incur charges.

------------------------------------------------------------------------

# 39. Can I Resize a Flex Instance?

Generally, EC2 instance type changes follow normal EC2 rules.

The important distinction is:

``` text
Resize
≠
CPU burst
```

If the workload permanently grows beyond the suitable Flex size, resize
or scale out.

Do not depend on Flex bursting as a permanent replacement for proper
capacity planning.

------------------------------------------------------------------------

# 40. Common Misconceptions

## Misconception 1

> Flex means CPU is capped at 40%.

**False.**

40% is the baseline. Flex can burst above it.

------------------------------------------------------------------------

## Misconception 2

> Flex uses CPU credits like T3.

**False.**

T-family burstable instances use CPU credits. Flex uses a different
performance model.

------------------------------------------------------------------------

## Misconception 3

> Going above 40% immediately creates a bill.

**False.**

Flex is not billed like T-family Unlimited CPU-credit usage.

------------------------------------------------------------------------

## Misconception 4

> Flex gives 100% CPU forever.

**False.**

AWS documents up to 100% CPU performance for 95% of the time over a
24-hour window, and sustained high CPU can gradually reduce maximum
burst throughput.

------------------------------------------------------------------------

## Misconception 5

> Flex is always cheaper than fixed.

**Not necessarily in total cost.**

The instance rate may be lower, but the correct choice depends on:

-   performance
-   scaling
-   workload behavior
-   architecture
-   application SLOs
-   Region
-   OS
-   storage
-   networking
-   purchasing model

------------------------------------------------------------------------

## Misconception 6

> Flex means all resources are flexible.

**False.**

The Flex model is primarily about CPU performance. Network, EBS, memory,
and storage capabilities remain defined by the instance type.

------------------------------------------------------------------------

# 41. Flex vs T3 vs M8i Example

Suppose you need:

``` text
8 vCPU
32 GiB RAM
```

Possible choices could include different instance families depending on
workload.

### T-family

Best when:

``` text
small/medium
variable CPU
very cost-sensitive
```

### M8i-flex

Best when:

``` text
modern production workload
needs 8 vCPU / 32 GiB class resources
CPU usually below maximum
occasional CPU bursts
```

### M8i

Best when:

``` text
same general resource class
but CPU can be continuously high
predictable sustained performance required
```

This is the core decision.

------------------------------------------------------------------------

# 42. A Practical Production Example

Imagine an API server:

``` text
Instance:
m8i-flex.2xlarge

vCPU:
8

RAM:
32 GiB
```

CPU pattern:

``` text
00:00 → 15%
04:00 → 10%
08:00 → 30%
10:00 → 45%
12:00 → 70%
14:00 → 90%
16:00 → 40%
18:00 → 25%
22:00 → 15%
```

This is a strong Flex candidate.

Why?

Because:

-   the workload is not continuously CPU-bound
-   it has predictable periods of higher demand
-   memory requirement is stable
-   CPU bursts are useful
-   lower-cost compute can improve economics

Now imagine:

``` text
00:00 → 85%
04:00 → 90%
08:00 → 95%
12:00 → 100%
16:00 → 95%
20:00 → 90%
```

That is a poor Flex profile.

Use a fixed-performance equivalent or redesign the scaling architecture.

------------------------------------------------------------------------

# 43. Production Checklist

Before moving a workload to Flex:

-   [ ] Confirm the exact Flex family is available in the target Region.
-   [ ] Confirm the required instance size exists.
-   [ ] Measure average CPU utilization.
-   [ ] Measure CPU peak utilization.
-   [ ] Measure peak duration.
-   [ ] Check memory utilization.
-   [ ] Check EBS performance.
-   [ ] Check network requirements.
-   [ ] Benchmark application latency.
-   [ ] Compare against the comparable non-Flex instance.
-   [ ] Compare against T-family where appropriate.
-   [ ] Calculate complete AWS cost, not only EC2 cost.
-   [ ] Configure CloudWatch monitoring.
-   [ ] Configure Auto Scaling where appropriate.
-   [ ] Define CPU/latency alarms based on workload behavior.
-   [ ] Test failure and replacement behavior.
-   [ ] Validate application performance under sustained CPU load.
-   [ ] Review the architecture after real production data is available.

------------------------------------------------------------------------

# 44. Troubleshooting Flex Performance

## Problem: CPU is high

Check:

``` text
CPUUtilization
```

Then ask:

``` text
Is this a short spike?
```

or:

``` text
Is CPU continuously high?
```

If continuously high:

``` text
Flex may be the wrong instance type
```

------------------------------------------------------------------------

## Problem: Application becomes slow

Do not automatically blame CPU.

Check:

``` text
CPU
Memory
EBS
Network
Database
Application thread pool
Connection pool
Load balancer
```

The bottleneck may be elsewhere.

------------------------------------------------------------------------

## Problem: CPU is below 40%, but application is slow

This can happen.

Why?

Because:

``` text
CPU utilization ≠ complete application health
```

Possible bottlenecks:

-   memory
-   EBS
-   network
-   database
-   locks
-   application code
-   external API
-   thread pool
-   connection pool

------------------------------------------------------------------------

# 45. Security

Flex does not fundamentally change the EC2 security model.

You still use:

``` text
VPC
Security Groups
NACLs
IAM
SSM
KMS
EBS encryption
CloudTrail
CloudWatch
GuardDuty
AWS Systems Manager
```

Flex is not a security feature.

It is a compute-performance/cost feature.

------------------------------------------------------------------------

# 46. Infrastructure-as-Code Considerations

For Terraform/CloudFormation/CDK, the instance type is simply part of
the configuration.

Example:

``` text
instance_type = "m8i-flex.large"
```

A good DevOps approach is to make the instance type configurable:

``` text
dev:
t3.medium

staging:
m8i-flex.large

production:
m8i-flex.large / m8i-flex.xlarge
```

Then benchmark before promoting.

------------------------------------------------------------------------

# 47. Cost Optimization Workflow

A practical workflow:

``` text
Step 1
Collect CloudWatch data
        ↓
Step 2
Find average CPU
        ↓
Step 3
Find CPU peaks
        ↓
Step 4
Find memory requirement
        ↓
Step 5
Find EBS/network bottleneck
        ↓
Step 6
Compare fixed vs Flex vs T
        ↓
Step 7
Benchmark
        ↓
Step 8
Deploy with IaC
        ↓
Step 9
Monitor
        ↓
Step 10
Recalculate cost
```

This is better than blindly changing instance types.

------------------------------------------------------------------------

# 48. How Flex Fits Into AWS Cost Optimization

Think of EC2 cost optimization as a stack:

``` text
                Cost Optimization
                       │
        ┌──────────────┼───────────────┐
        │              │               │
     Right-size     Purchase       Architecture
        │             model             │
        │              │               │
      Flex        Savings Plans      Auto Scaling
      Fixed       Spot               Containers
      T-family    On-Demand          Serverless
```

Flex is primarily a **right-sizing / price-performance optimization
tool**.

------------------------------------------------------------------------

# 49. Flex Does Not Replace Auto Scaling

These solve different problems.

### Flex

Optimizes:

``` text
CPU efficiency inside an instance
```

### Auto Scaling

Optimizes:

``` text
number of instances
```

Together:

``` text
Flex instance
     +
Auto Scaling
     =
better elasticity
```

------------------------------------------------------------------------

# 50. Flex Does Not Replace Serverless

Lambda and Flex solve different problems.

Lambda:

``` text
function-level execution
+
automatic scaling
+
pay-per-use
```

Flex:

``` text
full EC2 virtual server
+
OS control
+
long-running process
+
custom software
```

Choose based on workload architecture, not simply price.

------------------------------------------------------------------------

# 51. Flex Does Not Mean "Serverless CPU"

Flex is still a normal EC2 instance.

You still manage:

-   OS
-   packages
-   patching
-   agents
-   security
-   networking
-   storage
-   application runtime

unless those responsibilities are handled by another service.

------------------------------------------------------------------------

# 52. Flex and Container Platforms

Flex can be used as infrastructure for:

-   ECS
-   EKS
-   self-managed Kubernetes
-   container hosts

The important point is:

``` text
Container orchestration
       +
Flex compute
```

is a perfectly normal architecture.

------------------------------------------------------------------------

# 53. Flex and Spot --- They Can Be Combined

Flex describes:

``` text
performance model
```

Spot describes:

``` text
purchase model
```

Therefore these are not mutually exclusive concepts.

Conceptually:

``` text
Instance type:
m8i-flex.large

Purchase option:
Spot
```

can be used where the instance type and Region support it.

This can provide another layer of cost optimization, but the workload
must tolerate Spot interruption.

------------------------------------------------------------------------

# 54. Flex and Savings Plans

Again:

``` text
Flex = instance performance model
Savings Plan = purchasing commitment
```

They solve different problems.

A production workload can potentially use:

``` text
m8i-flex.xlarge
+
Savings Plan
```

if the usage pattern and commitment economics make sense.

------------------------------------------------------------------------

# 55. Region Availability Matters

Flex instance availability is **not automatically identical across every
AWS Region and Availability Zone**.

Before deployment:

``` text
Check:
1. Region
2. Availability Zone capacity
3. Instance type availability
4. On-Demand capacity
5. Spot availability if using Spot
6. Service quota
```

The EC2 instance-type catalog changes over time, so do not rely on an
old blog post or old architecture document.

------------------------------------------------------------------------

# 56. Capacity vs Performance

Another important distinction:

``` text
Instance availability
≠
CPU performance
```

You can have:

``` text
m8i-flex.4xlarge available
```

but still have:

``` text
application performance problem
```

if the workload requires sustained CPU.

Likewise, an instance can have enough CPU but insufficient:

``` text
RAM
EBS
network
```

Always size the complete resource profile.

------------------------------------------------------------------------

# 57. Best Use Cases

Flex is especially attractive for:

### Strong candidates

-   web servers
-   application servers
-   APIs
-   microservices
-   moderate databases
-   enterprise applications
-   virtual desktops
-   caches
-   Kafka
-   Elasticsearch
-   batch processing
-   development environments
-   staging
-   CI/CD infrastructure
-   variable CPU workloads

### Conditional candidates

-   production databases
-   CPU-based inference
-   analytics
-   data processing
-   large microservices

Benchmark carefully.

### Poor candidates

-   sustained 90--100% CPU
-   HPC requiring continuous maximum CPU
-   continuous video encoding
-   workloads needing the largest instance sizes
-   workloads requiring sustained maximum network/EBS throughput
-   highly CPU-bound batch processing where maximum throughput is
    continuously required

------------------------------------------------------------------------

# 58. The Most Important Difference in One Table

  -------------------------------------------------------------------------
  Characteristic    Fixed             Flex                Burstable T
  ----------------- ----------------- ------------------- -----------------
  Full CPU          Yes               Not the design goal Not the design
  continuously                                            goal

  Baseline          Full performance  40%                 Size dependent

  Burst             Not required      Yes                 Yes

  CPU credits       No                No                  Yes

  Unlimited CPU     No                No                  Yes, when
  credit charges                                          applicable

  95%/24-hour Flex  No                Yes                 No
  model                                                   

  Best for          Yes               No                  No
  sustained CPU                                           

  Best for variable Yes, but may cost Yes                 Yes
  CPU               more                                  

  Cost optimization Performance       Price/performance   Very low-cost
  target                                                  variable
                                                          workloads
  -------------------------------------------------------------------------

------------------------------------------------------------------------

# 59. Golden Rules

Remember these rules:

### Rule 1

> **Flex is not T3.**

------------------------------------------------------------------------

### Rule 2

> **40% is a baseline, not a CPU cap.**

------------------------------------------------------------------------

### Rule 3

> **Flex does not use the normal T-family CPU-credit model.**

------------------------------------------------------------------------

### Rule 4

> **95% of the time over 24 hours does not mean 95% CPU credits.**

------------------------------------------------------------------------

### Rule 5

> **High sustained CPU is a signal to evaluate the fixed-performance
> equivalent.**

------------------------------------------------------------------------

### Rule 6

> **Do not evaluate EC2 only by CPU.**

Check:

``` text
CPU + RAM + EBS + Network + latency + cost
```

------------------------------------------------------------------------

### Rule 7

> **Flex is an instance type/performance model, not a separate AWS
> service.**

------------------------------------------------------------------------

### Rule 8

> **Flex + Auto Scaling can be a very strong production architecture for
> variable workloads.**

------------------------------------------------------------------------

# 60. Interview-Ready Answer

If someone asks:

> "What are AWS EC2 Flex instances?"

Answer:

> **Amazon EC2 Flex instances are lower-priced variants of selected
> modern EC2 instance families designed for workloads that don't
> continuously use all available CPU resources. They provide a reliable
> 40% CPU baseline and can burst up to 100% CPU performance for 95% of
> the time over a 24-hour window. Unlike T-family burstable instances,
> Flex does not use the traditional CPU-credit model. Flex is best for
> variable general-purpose workloads where you want better
> price/performance, while the comparable non-Flex fixed-performance
> instance is better for sustained high CPU workloads.**

------------------------------------------------------------------------

# 61. Final Mental Model

Keep this diagram in your head:

``` text
                         EC2
                          │
             ┌────────────┼─────────────┐
             │            │             │
             ▼            ▼             ▼
          Fixed        Flex          Burstable
             │            │             │
             │            │             │
        Full CPU      40% baseline   CPU baseline
        anytime          │          + CPU credits
             │            │             │
             │         burst            │
             │            │             │
             │       up to 100%         │
             │            │             │
             │       95% / 24h          │
             │            │             │
             ▼            ▼             ▼
       Sustained      Variable        Small/
       high CPU       modern          medium
       workloads      workloads       variable
```

The simplest way to remember the three:

``` text
FIXED
"Give me full CPU whenever I need it."

FLEX
"I don't need full CPU all the time, but I want a modern,
larger EC2 instance that can burst when demand increases."

BURSTABLE
"I want a very economical instance and I'm okay with
the CPU-credit model."
```

------------------------------------------------------------------------

# 62. Final Recommendation

For a new workload, do not ask:

> "Should I use Flex?"

Ask:

``` text
What is my workload's CPU pattern?
What memory do I need?
What network do I need?
What EBS performance do I need?
How long do CPU peaks last?
What latency/SLO do I need?
How much can the workload scale horizontally?
What is the complete monthly cost?
```

Then compare:

``` text
T-family
vs
Flex
vs
Fixed-performance
vs
Specialized instance
```

and benchmark the real application.

**That is the correct DevOps approach: measure → benchmark → right-size
→ monitor → optimize.**

------------------------------------------------------------------------

# 63. AWS Sources

The following official AWS documentation/pages were used for this guide:

1.  Amazon EC2 Instance Types\
    https://docs.aws.amazon.com/ec2/latest/instancetypes/instance-types.html

2.  Amazon EC2 Instance Types Guide\
    https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-types.html

3.  Amazon EC2 On-Demand Pricing\
    https://aws.amazon.com/ec2/pricing/on-demand/

4.  Amazon EC2 Pricing\
    https://aws.amazon.com/ec2/pricing/

5.  Amazon EC2 Burstable Performance / CPU Credits\
    https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-credits-baseline-concepts.html

6.  Amazon EC2 M8i and M8i-flex\
    https://aws.amazon.com/ec2/instance-types/m8i/

7.  Amazon EC2 C7i and C7i-flex\
    https://aws.amazon.com/ec2/instance-types/c7i/

8.  Amazon EC2 Compute Optimized Instances\
    https://aws.amazon.com/ec2/instance-types/compute-optimized/

9.  Amazon EC2 R8i and R8i-flex\
    https://aws.amazon.com/ec2/instance-types/r8i/

10. Amazon EC2 Memory Optimized Instances\
    https://aws.amazon.com/ec2/instance-types/memory-optimized/

11. Amazon EC2 FAQ --- Flex Instances\
    https://aws.amazon.com/ec2/faqs/

------------------------------------------------------------------------

# 64. Important Accuracy Note

AWS continuously adds new EC2 families, sizes, Regions, and pricing
options.

Therefore, for production design:

-   verify the exact instance type in the target Region
-   verify current On-Demand pricing
-   verify Spot availability if applicable
-   verify service quotas
-   verify AMI/OS support
-   benchmark the workload

This document intentionally focuses on the **Flex architecture and
behavior model**, while exact prices and regional availability should be
treated as dynamic AWS data.
