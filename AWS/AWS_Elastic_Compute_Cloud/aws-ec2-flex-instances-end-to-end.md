# Amazon EC2 Flex Instances — End-to-End Guide

> **Status:** Revised September 16, 2026
> **Scope:** What Flex instances are, how the performance model works, families and sizes, limits, cost, selection method, deployment, monitoring, and troubleshooting.

---

## 1. Executive Summary

Amazon EC2 **Flex instances** are lower-priced variants of selected modern EC2 instance families, designed for workloads that need the resources of a normal EC2 instance but **do not continuously use 100% of the available CPU**.

The model:

- Flex instances provide a reliable CPU baseline of **40%**.
- When a workload needs more, they can **exceed the baseline and deliver up to 100% CPU performance for 95% of the time over a 24-hour window**.
- Flex instances running consistently above the baseline for long periods **may see a gradual reduction in maximum burst CPU throughput**.
- Flex is **not** the T-family burstable model and does **not** use CPU credits.

**Price positioning:** on the 8th-generation families, Flex variants are priced roughly **5% below the comparable non-Flex instance** (about 5% better price-performance). The 7th-generation Flex variants were positioned at up to **19% better price-performance than the comparable previous-generation instance**. Verify current rates in your Region — this is the number that decides whether Flex is worth the constraints.

Current Flex families:

| Family    | Category          | Processor                    |
|-----------|-------------------|------------------------------|
| C7i-flex  | Compute optimized | 4th Gen Intel Xeon Scalable  |
| C8i-flex  | Compute optimized | Custom Intel Xeon 6          |
| M7i-flex  | General purpose   | 4th Gen Intel Xeon Scalable  |
| M8i-flex  | General purpose   | Custom Intel Xeon 6          |
| R8i-flex  | Memory optimized  | Custom Intel Xeon 6          |

All Flex families offer **large through 16xlarge** — up to **64 vCPUs** and, on R8i-flex, up to **512 GiB** of memory.

**One-line mental model:**

> **Fixed** = full CPU whenever required
> **Burstable (T)** = credit-governed baseline and burst
> **Flex** = lower-cost instance with a reliable 40% baseline and time-based burst capability

---

## 2. Why AWS Introduced Flex

A common EC2 sizing problem is over-provisioning. Suppose an application needs 8 vCPUs and 32 GiB RAM, spikes to full CPU occasionally, but normally sits at 15–30%. A fixed-performance instance gives full CPU all the time — excellent performance, but you pay for capacity that is mostly idle.

Before Flex the choice was binary: a fixed-performance instance at full price, or a T-family burstable instance with CPU credits (which tops out at smaller sizes and brings credit management with it). Flex is the third option: modern instance families, common sizes up to 16xlarge, priced for workloads that don't saturate CPU.

---

## 3. What Exactly Is a Flex Instance?

A Flex instance is an ordinary EC2 instance with:

- a fixed amount of vCPU and memory
- defined network and EBS bandwidth
- a **40% CPU baseline**
- the ability to exceed that baseline
- no CPU-credit accounting

The critical point:

> **40% is a baseline, not a hard CPU cap.**

Launching an `m8i-flex.2xlarge` does not mean "you may only use 40% CPU." It means the instance delivers 40% CPU performance reliably and can go higher, subject to the Flex performance model.

Flex is also **not a separate service**. It is a performance characteristic of specific instance types:

```text
Amazon EC2
   └── Instance types
        ├── Fixed-performance families
        ├── Burstable T families
        └── Flex variants
```

---

## 4. The 40% Baseline — How to Reason About It

```text
0% ───────────── 40% ─────────────────────────── 100%
       baseline                 burst range
```

At or below ~40%, the workload is inside the guaranteed baseline. Above it, the instance uses burst capability.

The 40% figure refers to **CPU performance, not memory** — memory is always fully available.

For capacity planning, the rough arithmetic is fine as a first approximation:

```text
m8i-flex.2xlarge = 8 vCPU
40% baseline ≈ 3.2 vCPUs of sustained work
```

CloudWatch `CPUUtilization` is reported instance-wide, so this approximation lines up with what you actually measure. Where it breaks down is **single-threaded hot spots**: one process pinning one vCPU to 100% while the others idle shows up as ~12.5% instance CPU, well under baseline, yet that thread can still be the bottleneck. Always pair CPU percentage with application-level latency.

---

## 5. How the Burst Model Works

AWS describes Flex as providing up to 100% CPU performance for 95% of the time over a 24-hour window.

What this is **not**:

- It is not a bucket of credits you spend and replenish.
- There is no credit balance metric to watch, no Standard/Unlimited mode, and no surplus-credit charge.
- It is not a published, defined "degraded 5% window" you can schedule around. AWS does not document the mechanism, only the outcome.

What it means practically: if your workload bursts often but intermittently, burst capacity is there. If your workload sits above baseline for hours at a stretch, AWS states you may see a **gradual reduction in maximum burst CPU throughput** — at which point you are on the wrong instance type.

```text
Low/normal CPU  → operate at or below the 40% baseline
Traffic spike   → burst above baseline, up to full CPU
Sustained high  → maximum burst capability can gradually reduce
```

---

## 6. Flex vs Fixed vs Burstable

| Characteristic              | Fixed             | Flex                          | Burstable (T)                         |
|-----------------------------|-------------------|-------------------------------|---------------------------------------|
| Baseline                    | Full performance  | 40%                           | Size dependent (e.g. 10–40%)          |
| Burst mechanism             | N/A               | 95% / 24-hour model           | CPU credits                           |
| Credit balance to manage    | No                | No                            | Yes                                   |
| Surplus CPU charges         | No                | No                            | Yes, in Unlimited mode                |
| Can sustain full CPU        | Yes               | No — not the design goal      | Yes, in Unlimited mode, for a fee     |
| Size range                  | Up to 96xlarge + metal | large – 16xlarge         | nano – 2xlarge (family dependent)     |
| Best for                    | Sustained CPU     | Variable CPU, modern sizes    | Small/medium, very cost-sensitive     |

An important correction to a common comparison: a burstable instance configured as **unlimited can sustain high CPU utilization for any period of time**. The hourly price covers spikes as long as the 24-hour average stays at or below baseline; beyond that you pay a flat rate per vCPU-hour. The T-family constraint is therefore **cost**, not capability. Flex has no such escape hatch — sustained high CPU is a signal to move to the fixed-performance equivalent.

### Decision tree

```text
Start
 ├── Is CPU continuously high?           → YES → Fixed-performance instance
 ├── Small workload, very cost sensitive? → YES → T3 / T4g
 ├── Need modern family, more RAM/vCPU,
 │   CPU usually below maximum?           → YES → Flex
 └── Otherwise → benchmark Flex vs fixed vs burstable
```

---

## 7. Families and Sizes

All Flex families share the same size ladder: large, xlarge, 2xlarge, 4xlarge, 8xlarge, 12xlarge, 16xlarge.

### 7.1 C7i-flex and C8i-flex — compute optimized (2 GiB per vCPU)

| Size       | vCPU | Memory  |
|------------|------|---------|
| large      | 2    | 4 GiB   |
| xlarge     | 4    | 8 GiB   |
| 2xlarge    | 8    | 16 GiB  |
| 4xlarge    | 16   | 32 GiB  |
| 8xlarge    | 32   | 64 GiB  |
| 12xlarge   | 48   | 96 GiB  |
| 16xlarge   | 64   | 128 GiB |

Typical workloads: web and application servers, databases, caches, Apache Kafka, Elasticsearch, batch processing, distributed analytics, CPU-based inference.

C7i-flex runs on custom 4th Gen Intel Xeon Scalable processors. C8i-flex runs on custom Intel Xeon 6 processors and is roughly 20% faster than C7i-flex.

### 7.2 M7i-flex and M8i-flex — general purpose (4 GiB per vCPU)

| Size       | vCPU | Memory  |
|------------|------|---------|
| large      | 2    | 8 GiB   |
| xlarge     | 4    | 16 GiB  |
| 2xlarge    | 8    | 32 GiB  |
| 4xlarge    | 16   | 64 GiB  |
| 8xlarge    | 32   | 128 GiB |
| 12xlarge   | 48   | 192 GiB |
| 16xlarge   | 64   | 256 GiB |

Typical workloads: web applications, application servers, microservices, small and medium data stores, virtual desktops, enterprise applications.

Note: the 12xlarge and 16xlarge sizes were added to C7i-flex and M7i-flex in January 2025 and rolled out Region by Region — they are not everywhere the smaller sizes are.

### 7.3 R8i-flex — memory optimized (8 GiB per vCPU)

| Size       | vCPU | Memory  |
|------------|------|---------|
| large      | 2    | 16 GiB  |
| xlarge     | 4    | 32 GiB  |
| 2xlarge    | 8    | 64 GiB  |
| 4xlarge    | 16   | 128 GiB |
| 8xlarge    | 32   | 256 GiB |
| 12xlarge   | 48   | 384 GiB |
| 16xlarge   | 64   | 512 GiB |

Typical workloads: memory-heavy databases, Redis/Memcached, in-memory and real-time analytics, enterprise applications.

R8i-flex is the **first memory-optimized Flex family** — there is no R7i-flex.

### 7.4 Reading the name

```text
m8i-flex.4xlarge
│ │ │  │    └── size
│ │ │  └─────── Flex performance model
│ │ └────────── Intel
│ └──────────── generation
└────────────── general purpose family
```

---

## 8. What Flex Does Not Give You

This is the section most Flex guides skip, and it drives more instance-type decisions than the CPU model does.

- **No bare metal and no `.metal` sizes.** The non-Flex families offer 13 sizes including two bare metal and a 96xlarge; Flex stops at 16xlarge.
- **EBS-only storage.** No local NVMe instance store. If you need it, look at the `d` variants of the non-Flex family.
- **Capped and flat network/EBS bandwidth.** On C7i-flex and M7i-flex, every size from large to 8xlarge is rated up to 12.5 Gbps network and up to 10 Gbps EBS — bandwidth does **not** scale with instance size the way it does on non-Flex C7i/M7i. Sizing up for more throughput does not work on these families. (The 8i generation adds a configurable split between network and EBS bandwidth; verify the exact figures for the family and size you intend to use.)
- **Intel only.** There are no Graviton or AMD Flex variants.
- **Regional availability lags.** The 8i-flex families launched in a small set of Regions and are still expanding. Always check availability before designing around them.

If any of these constraints bind, the comparable non-Flex instance is the answer — not a larger Flex size.

---

## 9. Launching a Flex Instance

There is no "enable Flex" switch. The instance type name is the whole configuration.

### Console

EC2 → Instances → Launch instance → choose AMI → choose instance type (`m8i-flex.large`, `c8i-flex.2xlarge`, …) → networking → storage → IAM → security group → launch.

### CLI

```bash
aws ec2 run-instances \
  --image-id ami-xxxxxxxxxxxxxxxxx \
  --instance-type m8i-flex.large \
  --subnet-id subnet-xxxxxxxx \
  --security-group-ids sg-xxxxxxxx \
  --iam-instance-profile Name=MyEC2Role \
  --key-name my-key
```

### Terraform

```hcl
resource "aws_instance" "flex" {
  ami           = "ami-xxxxxxxxxxxxxxxxx"
  instance_type = "m8i-flex.large"
  subnet_id     = aws_subnet.private.id

  vpc_security_group_ids = [aws_security_group.app.id]
  iam_instance_profile   = aws_iam_instance_profile.ec2.name

  tags = {
    Name = "flex-app-server"
  }
}
```

Make the instance type a variable so you can promote through environments and benchmark before committing:

```text
dev        → t3.medium
staging    → m8i-flex.large
production → m8i-flex.large / m8i-flex.xlarge
```

Lifecycle behaviour (start, stop, reboot, terminate, instance-type modification) is standard EC2. Stopping ends compute billing; attached EBS volumes keep billing.

---

## 10. Billing

Flex instances are billed as ordinary EC2 compute. Bursting above 40% does **not** create a credit charge, because there are no credits.

```text
Compute cost = instance rate × running duration
```

On-Demand instances running Linux and Windows are billed by the second with a 60-second minimum; some commercial and Marketplace AMIs still bill by the hour, so confirm for your OS.

The instance rate is not the bill. Budget for:

```text
Total ≈ EC2 compute
      + EBS volumes, provisioned IOPS/throughput, snapshots
      + public IPv4 addresses
      + data transfer (internet, cross-AZ, cross-Region, NAT Gateway)
      + CloudWatch and logging
      + load balancers
      + OS/licensing charges
```

Two things to keep straight:

- **High CPU creates no CPU-credit surcharge on Flex, but a busier workload still generates more data transfer, more EBS I/O, and more logs.**
- **The network bandwidth in an instance spec is capacity, not a data-transfer allowance.**

Purchasing models are orthogonal to the performance model. `m8i-flex.xlarge` can be On-Demand, covered by a Savings Plan, or run as Spot where the type and Region support it. Flex describes *performance*; Spot and Savings Plans describe *purchasing*. They combine.

---

## 11. Choosing Correctly

Do not select on "Flex is cheaper." Measure, then benchmark:

1. Average CPU
2. Peak CPU
3. Duration of peaks
4. Memory utilisation
5. Network utilisation
6. EBS throughput and IOPS
7. Application latency against your SLO
8. Error rate and throughput
9. Complete monthly cost, not the EC2 line item

A Flex-friendly profile:

```text
Average CPU   = 25%
Peak CPU      = 90%
Peak duration = 10 minutes
RAM           = 20 GiB
```

A fixed-performance profile:

```text
Average CPU   = 75%
Peak CPU      = 100%
Peak duration = 8 hours
```

### Worked example — an API server on `m8i-flex.2xlarge` (8 vCPU, 32 GiB)

```text
00:00 → 15%   10:00 → 45%   18:00 → 25%
04:00 → 10%   12:00 → 70%   22:00 → 15%
08:00 → 30%   14:00 → 90%
```

Strong candidate: not continuously CPU-bound, predictable demand periods, stable memory, useful short bursts.

```text
00:00 → 85%   08:00 → 95%   16:00 → 95%
04:00 → 90%   12:00 → 100%  20:00 → 90%
```

Poor candidate: use the non-Flex equivalent, or redesign to scale horizontally.

---

## 12. Workload Fit

**Strong candidates:** web servers, application servers, APIs, microservices, moderate databases, caches, Kafka, Elasticsearch, virtual desktops, batch processing, CI/CD and automation servers, monitoring and internal tools, dev and staging environments.

**Conditional — benchmark first:** production databases, CPU-based inference, analytics and data processing, large microservices, CI workers.

**Poor candidates:** sustained 90–100% CPU, HPC needing continuous maximum CPU, continuous video encoding, anything needing the largest instance sizes, and anything needing sustained maximum network or EBS throughput.

A note on AI/ML: Flex is a CPU instance, not an AI instance. It can serve inference APIs, preprocessing, feature pipelines, and orchestration. GPU training, large GPU inference, and sustained CPU-heavy ML belong on specialised families.

---

## 13. Architecture Patterns

### Flex + Auto Scaling

The two solve different problems and compose well: Auto Scaling optimises the **number of instances**, Flex optimises **cost per instance for a variable CPU profile**.

```text
                 ALB
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
      Flex      Flex      Flex
       25%       30%       20%
```

Three smaller Flex instances behind a load balancer often beat one oversized fixed instance on both cost and resilience. During a spike, Auto Scaling adds capacity while each instance bursts to cover the gap.

### Flex + containers

Flex works as ECS/EKS node capacity wherever the type is available. The question is not "does Kubernetes support Flex" but "does my pod mix have a Flex-friendly CPU profile." Check CPU requests and limits, pod density, daemonset overhead, node utilisation, and latency SLOs.

### What Flex is not

- **Not high availability.** HA comes from multiple AZs, Auto Scaling, load balancing, health checks, and stateless design.
- **Not serverless.** You still own the OS, patching, agents, and runtime.
- **Not a security control.** The VPC, security group, NACL, IAM, KMS, SSM, CloudTrail, and GuardDuty model is unchanged.
- **Not a substitute for capacity planning.** If the workload permanently outgrows the size, resize or scale out.

---

## 14. Monitoring

Track at minimum:

- **EC2:** `CPUUtilization`, `NetworkIn`/`NetworkOut`, `NetworkPacketsIn`/`Out`, `DiskReadOps`/`DiskWriteOps`, `DiskReadBytes`/`DiskWriteBytes`, EBS burst and throughput metrics
- **Application:** request latency, requests per second, error rate, queue depth, worker and connection-pool utilisation, database latency

There is no credit-balance metric to watch — that concept does not exist on Flex.

### The 40% line is not an alarm threshold

```text
Wrong:   CPU > 40% → alarm → "Flex is failing"
Right:   CPU > 40% → instance is bursting → check duration and latency
```

A spike to 95% for two minutes is healthy. Sitting at 90–100% for hours is a different situation and a signal to re-evaluate the instance type. Alarm on **sustained** CPU above a workload-specific threshold, on latency against SLO, and on memory, EBS, and network saturation.

---
