# Amazon EBS Optimization -- Complete Guide

## Table of Contents

1.  Introduction
2.  What is EBS Optimization?
3.  How It Works
4.  Supported Instance Types
5.  When to Use EBS Optimization
6.  Real-World Use Cases
7.  Benefits
8.  Limitations
9.  Prerequisites
10. How to Check if EBS Optimization is Enabled
11. How to Enable EBS Optimization
12. CLI Commands
13. Performance Testing
14. CloudWatch Monitoring
15. Best Practices
16. Troubleshooting
17. Cost Considerations
18. Comparison
19. Interview Questions

------------------------------------------------------------------------

# 1. Introduction

Amazon Elastic Block Store (Amazon EBS) provides persistent block
storage for EC2 instances. EBS Optimization provides a **dedicated
storage network path** between an EC2 instance and its attached EBS
volumes, preventing storage traffic from competing with normal VPC or
internet traffic.

------------------------------------------------------------------------

# 2. What is EBS Optimization?

Without EBS Optimization, storage and network traffic share the same
bandwidth.

``` text
             Shared Network
Internet <---- EC2 ----> Amazon EBS
```

With EBS Optimization:

``` text
             Internet
                |
             EC2 Instance
            /            \
      VPC Network     Dedicated EBS Channel
                           |
                     Amazon EBS Volume
```

This dedicated channel delivers more consistent latency and throughput.

------------------------------------------------------------------------

# 3. How It Works

EBS-optimized instances reserve dedicated bandwidth for EBS I/O.

Storage traffic is isolated from:

-   Internet traffic
-   VPC communication
-   Application traffic
-   Load balancer traffic

This reduces contention and improves performance consistency.

------------------------------------------------------------------------

# 4. Supported Instance Types

Modern Nitro-based instance families are EBS-optimized by default.

Examples:

  -----------------------------------------------------------------------
  Family                                Status
  ------------------------------------- ---------------------------------
  t3/t4g                                Enabled by default

  m5/m6/m7                              Enabled by default

  c5/c6/c7                              Enabled by default

  r5/r6/r7                              Enabled by default

  i3/i4                                 Enabled by default

  Older generations                     May require manual enablement or
                                        may not support it
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 5. When Should You Use It?

Use EBS Optimization for:

-   Production workloads
-   Databases
-   Kubernetes worker nodes
-   CI/CD servers
-   Backup servers
-   Analytics platforms
-   High IOPS applications

Not usually necessary for:

-   Tiny development instances
-   Temporary lab instances
-   Very low disk activity workloads

------------------------------------------------------------------------

# 6. Real-World Use Cases

## Scenario 1 -- Database

MySQL/PostgreSQL with heavy transactions.

Benefit: - Lower latency - Higher IOPS - Stable performance

------------------------------------------------------------------------

## Scenario 2 -- Kubernetes (EKS)

Worker nodes using gp3/io2 Persistent Volumes.

Benefit: - Faster pod startup - Better PVC performance - Reduced storage
bottlenecks

------------------------------------------------------------------------

## Scenario 3 -- Jenkins

Large artifact storage.

Benefit: - Faster build execution - Faster workspace cleanup

------------------------------------------------------------------------

## Scenario 4 -- Backup Server

Large EBS snapshot creation.

Benefit: - Better snapshot throughput - Faster backup windows

------------------------------------------------------------------------

## Scenario 5 -- Log Processing

Splunk/ELK/OpenSearch.

Benefit: - Faster indexing - Better write throughput

------------------------------------------------------------------------

# 7. Benefits

-   Dedicated storage bandwidth
-   Lower latency
-   Predictable throughput
-   Improved IOPS consistency
-   Better application responsiveness
-   Improved backup performance
-   Reduced network contention
-   Better database performance

------------------------------------------------------------------------

# 8. Limitations

-   Does **not** increase CPU or memory performance.
-   Does **not** replace choosing the correct EBS volume type.
-   Throughput is still limited by the instance's documented EBS
    bandwidth.
-   Older instance types may not support it.
-   Poor application design cannot be fixed by enabling EBS
    Optimization.

------------------------------------------------------------------------

# 9. Prerequisites

-   EC2 instance that supports EBS Optimization
-   Attached EBS volume
-   IAM permissions for EC2 modifications (if required)
-   Instance stopped for older instance types when changing attributes

------------------------------------------------------------------------

# 10. Check Whether It Is Enabled

## AWS Console

EC2 → Instances → Select Instance → Details → **EBS Optimized**

## AWS CLI

``` bash
aws ec2 describe-instances \
--instance-ids i-xxxxxxxx \
--query "Reservations[].Instances[].EbsOptimized"
```

Expected:

``` json
[
  true
]
```

------------------------------------------------------------------------

# 11. Enable EBS Optimization

## During Launch

1.  Launch EC2.
2.  Expand **Advanced Details**.
3.  Enable **EBS Optimization** (if available).
4.  Launch the instance.

For modern Nitro instances, it is already enabled.

## Existing Instance

1.  Stop the instance.
2.  Modify instance type if required.
3.  Enable EBS Optimization (older supported types).
4.  Start the instance.

------------------------------------------------------------------------

# 12. Useful CLI Commands

Check:

``` bash
aws ec2 describe-instances \
--instance-ids i-123456789 \
--query "Reservations[*].Instances[*].EbsOptimized"
```

Modify (supported older types):

``` bash
aws ec2 modify-instance-attribute \
--instance-id i-123456789 \
--ebs-optimized
```

------------------------------------------------------------------------

# 13. Performance Testing

Install fio.

Ubuntu:

``` bash
sudo apt update
sudo apt install fio -y
```

Amazon Linux:

``` bash
sudo yum install fio -y
```

Sequential Write:

``` bash
fio --name=seqwrite \
--filename=/dev/nvme1n1 \
--rw=write \
--bs=1M \
--size=5G \
--runtime=60 \
--group_reporting
```

Random Read:

``` bash
fio --name=randread \
--filename=/dev/nvme1n1 \
--rw=randread \
--bs=4k \
--size=2G \
--runtime=60 \
--numjobs=4 \
--group_reporting
```

Observe: - IOPS - Throughput - Average latency - Queue depth

------------------------------------------------------------------------

# 14. CloudWatch Metrics

Volume Metrics

-   VolumeReadOps
-   VolumeWriteOps
-   VolumeReadBytes
-   VolumeWriteBytes
-   VolumeQueueLength
-   BurstBalance
-   VolumeIdleTime

EC2 Metrics

-   EBSReadBytes
-   EBSWriteBytes
-   EBSReadOps
-   EBSWriteOps

------------------------------------------------------------------------

# 15. Best Practices

-   Use gp3 instead of gp2 where possible.
-   Right-size IOPS and throughput.
-   Use io2 for mission-critical databases.
-   Prefer Nitro-based instances.
-   Monitor CloudWatch metrics regularly.
-   Keep EBS and instance sizes aligned with workload requirements.
-   Test using fio before production rollout.

------------------------------------------------------------------------

# 16. Troubleshooting

## EBS Optimized = False

-   Verify instance family supports it.
-   Upgrade to a newer instance type if needed.

## Low Performance

-   Check gp3 IOPS and throughput.
-   Ensure instance EBS bandwidth is not saturated.
-   Monitor VolumeQueueLength.
-   Check filesystem and application bottlenecks.

## High Latency

-   Confirm EBS volume type.
-   Check CloudWatch metrics.
-   Validate no CPU or memory bottlenecks.

------------------------------------------------------------------------

# 17. Cost Considerations

-   Current-generation Nitro instances generally include EBS
    Optimization at no extra charge.
-   You still pay for EBS storage, provisioned IOPS (where applicable),
    and throughput (gp3/io2 as configured).

------------------------------------------------------------------------

# 18. Comparison

  Feature                      Without   With
  ---------------------------- --------- --------
  Dedicated storage path       No        Yes
  Consistent latency           Lower     Higher
  Storage/network contention   Yes       No
  Predictable throughput       Lower     Higher

------------------------------------------------------------------------

# 19. Interview Questions

**Q: What is EBS Optimization?**

A dedicated storage channel between EC2 and EBS that improves storage
performance.

**Q: Does every EC2 support it?**

No. Most modern Nitro instances have it enabled by default.

**Q: Does it improve CPU performance?**

No. It only improves the storage path.

**Q: Is it useful for databases?**

Yes. It improves latency, throughput, and I/O consistency.

**Q: Should you always enable it?**

Use it whenever supported for production workloads. On modern instance
families it is already enabled.
