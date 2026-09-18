# AWS EC2 Instance Store — Complete End-to-End Guide

> **Level:** Beginner → Advanced
> **Service:** Amazon EC2
> **Main concept:** High-performance, temporary local block storage

---

# 1. What is EC2 Instance Store?

**EC2 Instance Store is temporary block-level storage physically attached to the EC2 host.**

It is also called:

* Ephemeral storage
* Local instance storage
* Local NVMe storage
* Instance store volume

Unlike EBS, Instance Store is **not an independent persistent storage service**.

### Simple example

```text
                    EC2
                     │
          ┌──────────┴──────────┐
          │                     │
         EBS              Instance Store
          │                     │
     Persistent             Temporary
     Storage                Storage
          │                     │
   Network attached        Local to host
```

---

# 2. The easiest way to remember it

### EBS

> "I need storage that survives my EC2 lifecycle."

### Instance Store

> "I need very fast local storage, and I can recreate the data if it disappears."

---

# 3. Why does AWS provide Instance Store?

Some workloads need:

* Very low latency
* Very high I/O performance
* Local storage
* Temporary storage
* Large scratch space
* Fast caching
* Temporary processing

For these workloads, persistent storage may not be necessary.

---

# 4. Instance Store Architecture

Conceptually:

```text
                    AWS Availability Zone
                             │
                             ▼
                    Physical EC2 Host
              ┌─────────────────────────┐
              │                         │
              │      EC2 Instance       │
              │                         │
              │    Application / OS     │
              │                         │
              │          │              │
              │          ▼              │
              │   Instance Store        │
              │      NVMe / SSD         │
              │                         │
              └─────────────────────────┘
```

The important point is:

> **Instance Store is physically local to the EC2 host.**

---

# 5. EBS vs Instance Store Architecture

## EBS

```text
Application
     │
     ▼
    EC2
     │
     │ Network
     ▼
    EBS
     │
     ▼
Persistent storage
```

## Instance Store

```text
Application
     │
     ▼
    EC2
     │
     ▼
Local Instance Store
     │
     ▼
Temporary storage
```

This local architecture is one reason Instance Store can provide very low latency.

---

# 6. Is Instance Store available on every EC2 instance?

**No.**

This is very important.

Not every EC2 instance type provides Instance Store.

Some instance types have:

```text
Instance storage: EBS only
```

Others provide:

```text
Instance storage:
1 × NVMe SSD
```

or multiple local NVMe devices.

Always check the specifications of the particular EC2 instance type.

---

# 7. What type of storage does Instance Store use?

Modern Instance Store configurations commonly use:

* NVMe SSD
* Local SSD storage

The exact storage technology and capacity depend on the EC2 instance type.

---

# 8. Is Instance Store persistent?

**No.**

Instance Store is considered **ephemeral storage**.

You should assume that data stored there can be lost when the instance lifecycle or underlying host changes.

Therefore:

```text
Important data
      ↓
Don't rely only on Instance Store
```

---

# 9. What happens during reboot?

A normal reboot of the same EC2 instance generally does **not** erase Instance Store data.

```text
Running
   │
   ▼
Reboot
   │
   ▼
Same EC2 instance
   │
   ▼
Instance Store data
   │
   └── Generally remains
```

However, never treat Instance Store as durable storage.

---

# 10. What happens during Stop → Start?

This is a very important interview question.

You should assume that Instance Store data is **lost when an instance is stopped**.

```text
Running
   │
   ▼
Stop
   │
   ▼
Start
   │
   ▼
Instance Store data
   │
   └── Lost
```

---

# 11. What happens during EC2 termination?

Instance Store data is lost.

```text
EC2
 │
 └── Instance Store
       │
       └── Data

Terminate EC2
       ↓
EC2 deleted
       ↓
Instance Store data deleted/lost
```

There is no:

```text
DeleteOnTermination = false
```

option for Instance Store like there is for EBS.

---

# 12. What happens if the underlying physical host fails?

The data in Instance Store can be lost.

This is one of the biggest differences from EBS.

```text
Physical Host
      │
      ├── EC2
      │
      └── Instance Store
             │
             └── Temporary data

Host failure
      ↓
Local Instance Store data
      ↓
Potentially lost
```

Therefore, applications using Instance Store should be designed to tolerate data loss.

---

# 13. Can Instance Store be detached?

**No.**

You cannot treat Instance Store like an EBS volume.

### EBS

```text
EC2-A
 │
 └── EBS
       ↓
   Detach
       ↓
   Attach
       ↓
EC2-B
```

### Instance Store

```text
EC2-A
 │
 └── Instance Store

Cannot detach and attach to EC2-B
```

---

# 14. Can you take an Instance Store snapshot?

**No.**

EBS supports snapshots.

Instance Store does not provide EBS-style snapshots.

```text
EBS
 │
 └── Snapshot ✅

Instance Store
 │
 └── Snapshot ❌
```

If you need to preserve data, copy it to durable storage such as:

* EBS
* S3
* Another appropriate persistent service

---

# 15. Can you back up Instance Store?

Not directly using EBS snapshots.

If the data is important, your application must copy it somewhere durable.

Example:

```text
Instance Store
      │
      │ Backup/copy
      ▼
     S3
```

---

# 16. Is Instance Store free?

There is generally **no separate storage-volume charge like EBS** for the Instance Store included with supported EC2 instance types.

However:

> **The EC2 instance itself is still charged normally.**

Conceptually:

```text
EC2 instance
     │
     ├── Compute → Charged
     │
     └── Included Instance Store
                    │
                    └── No separate EBS-style
                        storage-volume charge
```

The EC2 instance price depends on the instance type.

---

# 17. Why use Instance Store if EBS already exists?

This is the key question.

EBS gives you:

* Persistence
* Detach/attach
* Snapshots
* Backup capabilities
* Independent lifecycle

But Instance Store gives you:

* Local storage
* Very low latency
* High local I/O performance
* Temporary scratch space
* Fast caching

So the choice is not simply:

```text
EBS vs Instance Store
```

It is:

```text
Do I need persistence?
       │
       ├── YES → EBS / S3 / database service
       │
       └── NO
            │
            └── Do I need very fast local storage?
                    │
                    └── YES → Instance Store
```

---

# 18. Scenario 1 — Application Cache

Suppose you have a web application.

```text
                 Users
                   │
                   ▼
                  EC2
             ┌─────┴─────┐
             │           │
           EBS       Instance Store
             │           │
          App data      Cache
```

The cache can be recreated.

If the instance disappears:

```text
Cache → Lost ❌
```

The application rebuilds it.

### Why Instance Store?

Because:

* Cache is temporary
* Cache can be regenerated
* Fast local access is useful

---

# 19. Scenario 2 — Video Processing

Suppose you process large videos.

```text
                    S3
                     │
                Original video
                     │
                     ▼
                    EC2
                     │
                     ▼
              Instance Store
                     │
          ┌──────────┼──────────┐
          │          │          │
        Frames    Temp files  Processing
          │          │          │
          └──────────┼──────────┘
                     │
                     ▼
                 Final video
                     │
                     ▼
                    S3
```

The intermediate files don't need to survive.

### Instance Store is appropriate.

---

# 20. Scenario 3 — Temporary Log Processing

Suppose an application generates large log files.

```text
Application
     │
     ▼
Instance Store
     │
     ├── Raw logs
     ├── Temporary files
     └── Processing
     │
     ▼
Compressed logs
     │
     ▼
S3
```

Once the data reaches S3, the local copy can be deleted.

---

# 21. Scenario 4 — Data Processing

A data-processing application creates hundreds of GB of intermediate files.

```text
Input
  │
  ▼
EC2
  │
  ▼
Instance Store
  │
  ├── temp-1
  ├── temp-2
  ├── temp-3
  └── temp-4
  │
  ▼
Final result
  │
  ▼
S3
```

The temporary files don't need to survive.

Instance Store can be useful here.

---

# 22. Scenario 5 — Database

Suppose you run MySQL.

```text
EC2
 │
 └── MySQL
       │
       └── Instance Store
```

This is generally a poor choice if the data is the authoritative production database.

Why?

```text
Instance/host failure
        ↓
Local data can be lost
        ↓
Potential database loss
```

Use durable storage such as EBS or a managed database service, depending on the architecture.

---

# 23. Scenario 6 — User Uploads

Suppose users upload profile pictures.

Bad design:

```text
User
 │
 ▼
EC2
 │
 ▼
Instance Store
 │
 └── profile.jpg
```

If the instance disappears:

```text
profile.jpg → Lost
```

Better:

```text
User
 │
 ▼
Application
 │
 ▼
S3
 │
 └── profile.jpg
```

S3 is designed for durable object storage.

---

# 24. Scenario 7 — Scratch Space

Suppose an application needs 1 TB of temporary workspace.

```text
Application
     │
     ▼
Instance Store
     │
     └── 1 TB temporary workspace
```

The application processes data and deletes the temporary files afterward.

This is a classic Instance Store use case.

---

# 25. Scenario 8 — High-performance temporary workload

Imagine a workload requiring:

* Very low latency
* High IOPS
* Local processing
* Temporary data

Instance Store may be suitable.

```text
Application
     │
     ▼
Local NVMe Instance Store
     │
     ▼
High-speed processing
```

The important trade-off is:

```text
Performance ↑
Persistence ↓
```

---

# 26. Can Instance Store replace EBS?

**No, not generally.**

They solve different problems.

A typical EC2 architecture can use both:

```text
                    EC2
                     │
          ┌──────────┴──────────┐
          │                     │
         EBS              Instance Store
          │                     │
       Persistent          Temporary
       data                 data
          │                     │
       OS/App               Cache
                            Scratch
                            Processing
```

---

# 27. EBS + Instance Store Together

A production application could look like:

```text
                    EC2
                     │
          ┌──────────┴───────────┐
          │                      │
       EBS Volume          Instance Store
          │                      │
          │                      │
    ┌─────┴─────┐         ┌──────┴──────┐
    │            │         │             │
   OS       Application   Cache       Temp files
    │
 Persistent data
```

This is often more realistic than choosing only one.

---

# 28. EBS vs Instance Store — Complete Comparison

| Feature                    | EBS           | Instance Store                      |
| --------------------------- | ------------- | ------------------------------------ |
| Block storage               | ✅             | ✅                                    |
| Persistent                  | ✅             | ❌                                    |
| Local to physical host      | ❌             | ✅                                    |
| Network attached            | ✅             | ❌                                    |
| Survives reboot             | ✅             | Generally yes                        |
| Survives stop/start         | ✅             | ❌                                    |
| Survives termination        | ✅ If retained | ❌                                    |
| Detachable                  | ✅             | ❌                                    |
| Attach to another EC2       | ✅             | ❌                                    |
| EBS snapshots                | ✅             | ❌                                    |
| Backup                      | ✅             | Must copy data elsewhere             |
| Very low local latency      | Good          | **Excellent**                        |
| Temporary files             | ✅             | **Excellent**                        |
| Cache                       | ✅             | **Excellent**                        |
| Production persistent data  | ✅             | ❌                                    |
| Separate storage billing    | ✅             | Generally no separate volume charge  |
| Available on all EC2 types  | No            | No                                   |
| Lifecycle                   | Independent   | Tied to instance/host                |

---

# 29. Important EBS Concept — DeleteOnTermination

Do not confuse EBS with Instance Store.

EBS can have:

```text
DeleteOnTermination = true
```

Example:

```text
EC2
 │
 └── Root EBS
      DeleteOnTermination = true

Terminate EC2
      ↓
EBS deleted
```

Or:

```text
EC2
 │
 └── Data EBS
      DeleteOnTermination = false

Terminate EC2
      ↓
EBS remains
```

Instance Store doesn't work this way.

```text
Instance Store
      ↓
Ephemeral
      ↓
No DeleteOnTermination control
```

---

# 30. Important lifecycle table

| EC2 operation | EBS                                   | Instance Store          |
| -------------- | -------------------------------------- | ------------------------ |
| Reboot         | Data remains                           | Generally remains       |
| Stop           | Data remains                           | Data lost                |
| Start          | Data remains                           | New local storage state  |
| Terminate      | Depends on DeleteOnTermination         | Data lost                |
| Host failure   | EBS designed to persist independently  | Data can be lost         |
| Detach         | ✅                                       | ❌                        |

---

# 31. Instance Store and Auto Scaling

This is an important real-world concept.

Suppose you have:

```text
Auto Scaling Group
       │
 ┌─────┼─────┐
 ▼     ▼     ▼
EC2   EC2   EC2
 │     │     │
IS    IS    IS
```

Each instance has its own local Instance Store.

If Auto Scaling replaces one instance:

```text
Old EC2
   │
   └── Instance Store ❌

New EC2
   │
   └── New Instance Store
```

The new instance does not receive the old instance's local data.

Therefore, applications must not depend on Instance Store as the only copy of important data.

---

# 32. Instance Store in Auto Scaling Applications

Good:

```text
S3
 │
 ▼
EC2
 │
 └── Instance Store
       │
       └── Temporary processing
```

Bad:

```text
EC2
 │
 └── Instance Store
       │
       └── Only copy of production data
```

---

# 33. Instance Store and Load Balancers

Imagine:

```text
                 ALB
                  │
          ┌───────┼───────┐
          ▼       ▼       ▼
         EC2     EC2     EC2
          │       │       │
         IS      IS      IS
```

Each Instance Store is local to its own EC2.

Therefore:

```text
EC2-A Instance Store
        ≠
EC2-B Instance Store
```

If a user uploads a file to EC2-A's local disk, EC2-B cannot automatically see that file.

For shared persistent files, use an appropriate shared storage service such as S3 or EFS depending on the access pattern.

---

# 34. Instance Store and Containers

Instance Store can be useful as temporary local storage for container workloads.

Example:

```text
EC2
 │
 ├── Docker
 │    │
 │    └── Container
 │          │
 │          ▼
 │     Instance Store
 │
 └── EBS
```

Useful for:

* temporary container data
* build artifacts
* cache
* scratch space

But persistent application data should be stored separately.

---

# 35. How to identify Instance Store disks in Linux

SSH/Session Manager into the instance and run:

```bash
lsblk
```

Example:

```text
NAME        SIZE TYPE MOUNTPOINT
nvme0n1      30G disk
├─nvme0n1p1  30G part /
nvme1n1     475G disk
```

The additional NVMe device may be an Instance Store device, depending on the instance type and configuration.

---

# 36. Check the filesystem

You can use:

```bash
df -h
```

Example:

```text
Filesystem      Size  Used Avail Use%
/dev/root        30G   10G   20G  34%
/dev/nvme1n1    475G    1G  474G   1%
```

---

# 37. Check block devices

Useful commands:

```bash
lsblk
```

```bash
sudo fdisk -l
```

```bash
df -h
```

```bash
mount
```

For NVMe devices, you can also inspect:

```bash
sudo nvme list
```

if the `nvme-cli` package is installed.

---

# 38. Formatting an Instance Store volume

Suppose the device is:

```text
/dev/nvme1n1
```

You might format it:

```bash
sudo mkfs.ext4 /dev/nvme1n1
```

Then create a mount point:

```bash
sudo mkdir /data
```

Mount it:

```bash
sudo mount /dev/nvme1n1 /data
```

Check:

```bash
df -h
```

> Be extremely careful with `mkfs`. Formatting the wrong device can destroy data.

---

# 39. Mounting Instance Store automatically

If you want automatic mounting, you can configure `/etc/fstab`, but because Instance Store is ephemeral, you should design the boot process to tolerate the disk being absent or recreated.

A robust application should initialize the disk when the instance starts.

Conceptually:

```text
EC2 starts
   ↓
Check Instance Store
   ↓
Format if required
   ↓
Mount
   ↓
Start application
```

---

# 40. Instance Store and AMI

The AMI/root EBS volume and Instance Store are different concepts.

Example:

```text
AMI
 │
 └── Root filesystem
       ↓
      EBS

EC2 instance
 │
 └── Instance Store
       ↓
    Local temporary disk
```

When a new EC2 instance is launched from an AMI, the local Instance Store is not a persistent copy of the previous instance's local data.

---

# 41. Instance Store and EC2 Instance Type

Always check:

```text
Instance type
      ↓
Instance storage
      ↓
Capacity
      ↓
Number of local disks
      ↓
Storage technology
```

Do not assume:

> "Every `m`, `c`, or `r` instance has Instance Store."

Availability is specific to the instance type.

---

# 42. Instance Store capacity

Capacity varies by instance type.

You may see configurations such as:

```text
1 × local NVMe SSD
```

or:

```text
2 × local NVMe SSD
```

or larger local-storage configurations.

Always check the current AWS instance-type specification for the exact capacity.

---

# 43. Can you increase Instance Store size?

You generally cannot resize an Instance Store volume like an EBS volume.

If you need more capacity, you typically choose an appropriate EC2 instance type that provides the required local storage.

For example:

```text
Current instance
    ↓
500 GB local storage

Need
    ↓
1.5 TB local storage

Solution
    ↓
Choose an instance type with sufficient
local Instance Store capacity
```

---

# 44. Can you encrypt Instance Store?

Modern AWS EC2 local NVMe instance storage on supported instance types can provide encryption at the hardware level.

The exact behavior depends on the instance type and AWS implementation.

Do not confuse this with:

```text
EBS encryption using KMS
```

Instance Store and EBS have different storage architectures and encryption mechanisms.

---

# 45. Security consideration

Never assume temporary means unimportant.

Instance Store may contain:

* temporary customer information
* decrypted files
* processing data
* credentials accidentally written to disk
* logs
* cached data

Applications should avoid putting secrets or sensitive data unnecessarily on local storage.

Use appropriate:

* IAM
* encryption
* application-level security
* secure deletion practices
* access controls

---

# 46. Instance Store failure model

The biggest weakness is:

```text
Local storage
     ↓
Instance/host lifecycle
     ↓
Data can disappear
```

Therefore:

> **Design the application assuming Instance Store can be lost at any time.**

This is especially important in:

* Auto Scaling
* Kubernetes
* Spot Instances
* batch processing
* distributed systems

---

# 47. Instance Store + Spot Instances

This is a common combination.

Spot Instances can be interrupted.

If your workload uses:

```text
Spot EC2
   +
Instance Store
```

you should assume temporary local data can disappear.

Good workload:

```text
S3 input
  ↓
Spot EC2
  ↓
Instance Store
  ↓
Processing
  ↓
S3 output
```

If the Spot Instance is interrupted:

```text
Instance Store → Lost
```

The job can be restarted.

---

# 48. Instance Store + Distributed Applications

Instance Store works particularly well when data can be reconstructed.

For example:

```text
                    S3
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
        EC2-A      EC2-B      EC2-C
          │          │          │
         IS         IS         IS
          │          │          │
       Temp A      Temp B     Temp C
```

Each node can have its own temporary workspace.

The durable source of truth remains outside the Instance Store.

---

# 49. The most important design rule

Use:

```text
Instance Store
     ↓
Cache
Scratch
Temporary processing
Reproducible data
```

Don't use it as the only location for:

```text
Production database
Customer records
Only copy of files
Critical application state
Permanent backups
```

---

# 50. Common interview traps

## Trap 1

**"Instance Store survives reboot, so it's persistent."**

❌ Wrong.

Reboot persistence does not make it durable storage.

---

## Trap 2

**"EBS always survives termination."**

❌ Not necessarily.

Check:

```text
DeleteOnTermination
```

---

## Trap 3

**"Instance Store is cheaper, so I should use it instead of EBS."**

❌ Wrong reasoning.

Storage cost isn't the main decision.

The key question is:

```text
Do I need persistence?
```

---

## Trap 4

**"Instance Store is slower because it's temporary."**

❌ Wrong.

Instance Store can provide extremely high local performance.

---

## Trap 5

**"I can detach Instance Store and attach it to another EC2."**

❌ No.

That's an EBS capability.

---

## Trap 6

**"I can take an EBS snapshot of Instance Store."**

❌ No.

Instance Store doesn't support EBS snapshots.

---

# 51. Interview Scenario

### Question

> You have an EC2 application that processes 500 GB files. During processing, it creates hundreds of GB of temporary files. The final output is stored in S3. Which storage would you consider?

### Answer

```text
S3
 │
 ▼
EC2
 │
 ▼
Instance Store
 │
 ├── Temporary input
 ├── Intermediate files
 └── Processing
 │
 ▼
Final output
 │
 ▼
S3
```

Instance Store is suitable because:

1. Temporary data can be recreated.
2. Local storage can provide high performance.
3. The final durable result is stored in S3.
4. Losing the local temporary data doesn't mean losing the source of truth.

---

# 52. Interview Scenario — Database

### Question

> Your production MySQL database is running on EC2. Should you store the database on Instance Store?

### Answer

Generally, **not as the only durable copy**.

Use durable storage such as:

```text
EC2
 │
 └── EBS
       │
       └── Database
```

or use an appropriate managed database service.

---

# 53. Interview Scenario — Cache

### Question

> Your application needs a very fast cache. The cache can be rebuilt from the database. Can Instance Store be used?

### Answer

Yes.

```text
Database
   │
   ▼
Application
   │
   ▼
Instance Store
   │
   └── Cache
```

If the instance disappears:

```text
Cache → Lost
```

The application rebuilds it.

---

# 54. Interview Scenario — Auto Scaling

### Question

> You have 10 EC2 instances behind an ALB. Each instance stores uploaded images in Instance Store. Is this architecture safe?

### Answer

No.

Each instance has its own local storage:

```text
User
 │
 ▼
ALB
 │
 ├── EC2-A → image.jpg
 ├── EC2-B
 └── EC2-C
```

The next request may go to EC2-B.

EC2-B won't automatically have the file stored on EC2-A.

Use shared/durable storage such as S3 for uploaded objects.

---

# 55. Quick Decision Framework

Ask these three questions:

### Question 1

**Does the data need to survive the EC2 instance lifecycle?**

```text
YES → Persistent storage
NO  → Continue
```

### Question 2

**Can the data be recreated?**

```text
YES → Instance Store can be considered
NO  → Use durable storage
```

### Question 3

**Do I benefit from very fast local storage?**

```text
YES → Instance Store may be a good fit
NO  → Other storage may be simpler
```

---

# 56. Final Mental Model

Remember this architecture:

```text
                         EC2
                          │
             ┌────────────┴────────────┐
             │                         │
            EBS                  Instance Store
             │                         │
       Persistent                  Temporary
             │                         │
       Network-based                 Local
             │                         │
       Detachable                   Not detachable
             │                         │
       Snapshots                    No EBS snapshot
             │                         │
    Survives lifecycle          Lifecycle dependent
             │                         │
       Important data              Cache
       OS / application            Scratch
       Database                    Temp processing
                                  High-speed workspace
```

# 57. One-Line Definition

> **EC2 Instance Store is high-performance, physically local block storage provided with certain EC2 instance types, designed primarily for temporary data such as cache, scratch space, and intermediate processing where data loss is acceptable.**

# 58. One-Line EBS vs Instance Store

> **EBS = persistent, independently managed block storage; Instance Store = local, high-performance, ephemeral block storage tied to the EC2 instance/host.**

# 59. Must-Remember Points

Before an EC2 assessment, remember these:

* Instance Store = **ephemeral/local storage**
* Available only on **certain EC2 instance types**
* Commonly uses **local NVMe SSD**
* Designed for **high performance and low latency**
* Not intended for critical persistent data
* Data can be lost when the instance is stopped
* Data is lost when the instance is terminated
* Host failure can result in data loss
* Reboot generally preserves the data
* Cannot detach like EBS
* Cannot take EBS snapshots
* No `DeleteOnTermination` setting like EBS
* No separate EBS-style storage-volume charge for included Instance Store
* EC2 compute is still billed
* Excellent for cache
* Excellent for scratch space
* Excellent for temporary processing
* Useful with batch workloads
* Useful with some distributed workloads
* Useful with Spot Instances when data is reproducible
* Don't store the only copy of important data there
* Use EBS/S3/database services for durable data

---

# 60. Final Memory Trick

### **EBS**

```text
KEEP IT
↓
Persistent
↓
Detach
↓
Snapshot
↓
Backup
```

### **Instance Store**

```text
USE IT FAST
↓
Local
↓
Temporary
↓
Cache
↓
Scratch
↓
Processing
↓
Can disappear
```

> **If losing the data would be a disaster → don't make Instance Store your only storage.**
