# AWS DataSync Task Options --- Complete Guide

## 1. What is AWS DataSync?

**AWS DataSync** is a managed AWS service used to move and synchronize
file/object data between supported storage locations.

Typical examples:

``` text
On-Premises NFS/SMB
        |
        | AWS DataSync
        v
   Amazon S3 / EFS / FSx
```

The DataSync **Task** is where we define **how the transfer should
behave**.

A useful way to think about a DataSync task is:

> **What should I scan? What should I transfer? What should I verify?
> What should happen to destination files? What metadata should I
> preserve? How should I monitor it?**

------------------------------------------------------------------------

# 2. Current Task Configuration

The configuration shown in the AWS console is:

  Setting                Current value
  ---------------------- -----------------------------------------
  Contents to scan       **Everything**
  Excludes               **None**
  Transfer mode          **Transfer only data that has changed**
  Verification           **Verify only transferred data**
  Bandwidth limit        **Use available**
  Keep deleted files     **OFF**
  Overwrite files        **ON**
  Copy ownership         **ON**
  Copy permissions       **ON**
  Copy timestamps        **ON**
  Queueing               **ON**
  Schedule               **Not scheduled**
  Task report            **None**
  Log level              **Basic**
  CloudWatch log group   **/aws/datasync**

This configuration is designed primarily for a **source-authoritative
synchronization/mirror-style workload**.

------------------------------------------------------------------------

# 3. Overall DataSync Flow

``` text
                     SOURCE
                       |
                       v
              +----------------+
              | Contents scan  |
              |   Everything   |
              +-------+--------+
                      |
                      v
              +----------------+
              | Apply excludes |
              |     None       |
              +-------+--------+
                      |
                      v
              +----------------+
              | Compare source |
              | & destination  |
              +-------+--------+
                      |
              Changed data only
                      |
                      v
              +----------------+
              |    Transfer    |
              +-------+--------+
                      |
                      v
              +----------------+
              |    Verify      |
              | transferred    |
              |     data       |
              +-------+--------+
                      |
                      v
                 DESTINATION
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
      Ownership   Permissions  Timestamps
       preserved   preserved    preserved

Source-deleted files
        |
        v
Destination copy deleted
(Keep deleted files = OFF)

Changed source files
        |
        v
Destination copy overwritten
(Overwrite = ON)
```

------------------------------------------------------------------------

# 4. Contents to Scan

## What?

**Contents to scan** defines which source data DataSync considers for
the task.

Current value:

``` text
Everything
```

This means DataSync considers the complete source location.

### Example

Source:

``` text
/source
├── application/
│   ├── config.yml
│   └── app.jar
├── database/
│   └── backup.sql
├── images/
│   ├── logo.png
│   └── user.jpg
└── logs/
    └── application.log
```

With:

``` text
Contents to scan = Everything
```

all of these are considered.

------------------------------------------------------------------------

## Alternative: Specific files, objects, and folders

Instead of scanning everything, you can specify particular source paths.

Example:

``` text
/application
/database
```

Then:

``` text
application/  -> considered
database/     -> considered
images/       -> not selected
logs/         -> not selected
```

### When to use Specific?

Use it when the source contains a large amount of data but only certain
folders need to be transferred.

### Simple rule

``` text
Everything
    |
    +--> Full source synchronization

Specific
    |
    +--> Selected source data only
```

------------------------------------------------------------------------

# 5. Excludes

## What?

**Excludes** lets you tell DataSync:

> "Do not transfer these files, objects, or folders."

Current value:

``` text
None
```

So nothing is explicitly excluded.

------------------------------------------------------------------------

## Example

Suppose:

``` text
/source
├── application/
├── database/
├── logs/
├── temp/
└── cache/
```

You could exclude:

``` text
/temp
/cache
*.tmp
```

Then:

``` text
application/  -> transferred
database/     -> transferred
logs/         -> transferred
temp/         -> skipped
cache/        -> skipped
```

------------------------------------------------------------------------

## Why use excludes?

Common reasons:

-   Temporary files are not required.
-   Cache files are recreated automatically.
-   Large log files are stored somewhere else.
-   Certain directories contain sensitive or unnecessary data.
-   You want to reduce transfer size.

### Simple rule

> **Exclude = "Consider the source, but don't transfer matching data."**

------------------------------------------------------------------------

# 6. Transfer Mode

This is one of the most important DataSync settings.

Current value:

``` text
Transfer only data that has changed
```

The console provides two main choices:

1.  Transfer all data
2.  Transfer only data that has changed

------------------------------------------------------------------------

## 6.1 Transfer only data that has changed

### What?

DataSync transfers only data and metadata that differ between the source
and destination.

### Example

First execution:

``` text
SOURCE                  DESTINATION

A.txt  ----------------> A.txt
B.txt  ----------------> B.txt
C.txt  ----------------> C.txt
```

Later:

``` text
A.txt = unchanged
B.txt = changed
C.txt = unchanged
```

DataSync transfers the required changed data:

``` text
B.txt
```

instead of blindly copying everything again.

### Why?

It reduces:

-   network traffic
-   transfer time
-   unnecessary data movement
-   operational overhead

### Best use case

Recurring synchronization:

``` text
On-Prem
   |
   v
DataSync
   |
   v
S3/EFS/FSx
```

Run repeatedly and transfer only what changed.

------------------------------------------------------------------------

## 6.2 Transfer all data

### What?

DataSync copies all source content to the destination without comparing
existing destination content to determine what has changed.

Example:

``` text
Source = 10 TB
```

Run 1:

``` text
10 TB transferred
```

Run 2:

``` text
10 TB transferred
```

Run 3:

``` text
10 TB transferred
```

Even if only 100 GB changed, this mode does not use the normal
changed-data comparison behavior to reduce the transfer.

### When to use?

Useful when the requirement is essentially:

> "Copy the entire source dataset."

This can make sense for certain full-copy/migration workflows.

### Simple comparison

  -----------------------------------------------------------------------
  Transfer mode           Behavior                Typical use
  ----------------------- ----------------------- -----------------------
  **Transfer only data    Transfers differences   Recurring sync
  that has changed**                              

  **Transfer all data**   Copies all source       Full copy
                          content                 
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 7. Verification

Current value:

``` text
Verify only transferred data
```

DataSync performs integrity checks during transfers. Verification
controls additional verification after the transfer.

The console provides:

1.  Don't verify data after transfer
2.  Verify only transferred data
3.  Verify all data

------------------------------------------------------------------------

## 7.1 Don't verify data after transfer

### What?

DataSync checks data during transfer but does not perform additional
post-transfer verification.

Conceptually:

``` text
Transfer
   |
   v
Integrity check during transfer
   |
   v
Done
```

### Advantage

Less additional verification work.

### Use when

You accept the integrity checks performed during transfer and don't
require an additional verification pass.

------------------------------------------------------------------------

## 7.2 Verify only transferred data

### What?

At the end of the transfer, DataSync checks the data that was
transferred and confirms it is synchronized between source and
destination.

Example:

``` text
Total source data = 10 TB

Changed data = 200 GB

Transfer:
200 GB

Additional verification:
Transferred data
```

### Why?

You get an additional integrity/synchronization check without verifying
the entire dataset every time.

### Good choice for

-   Recurring synchronization
-   Large datasets
-   Normal production transfers
-   Incremental transfers

This is the current configuration.

------------------------------------------------------------------------

## 7.3 Verify all data

### What?

DataSync verifies the entire source and destination to check that the
locations are fully synchronized.

Conceptually:

``` text
SOURCE                  DESTINATION
10 TB                    10 TB
  |                         |
  +-------- COMPARE --------+
              |
              v
       Fully synchronized?
```

### Advantage

Maximum confidence in the synchronization state.

### Disadvantage

More scanning and processing, especially with very large datasets.

### When to use?

Use it when complete verification is more important than minimizing
additional verification work.

------------------------------------------------------------------------

# 8. Bandwidth Limit

Current value:

``` text
Use available
```

## What?

This controls the maximum network bandwidth DataSync can use.

With:

``` text
Use available
```

DataSync can use available network bandwidth rather than being limited
to a manually specified rate.

------------------------------------------------------------------------

## Example

Suppose:

``` text
Company WAN = 1 Gbps
```

and DataSync is running.

If you allow available bandwidth:

``` text
WAN
 |
 +---- DataSync
 |
 +---- Users
 |
 +---- Applications
```

DataSync can consume substantial available bandwidth.

------------------------------------------------------------------------

## Alternative: Set a bandwidth limit

Example:

``` text
DataSync limit = 100 Mbps
```

Now DataSync is deliberately limited.

### Why use a limit?

Suppose your WAN is shared:

``` text
                    1 Gbps WAN
                        |
        +---------------+---------------+
        |               |               |
     DataSync         Users           Apps
```

If DataSync consumes too much bandwidth, users and applications may
suffer.

So you can set:

``` text
DataSync = 100 Mbps
```

to protect other traffic.

### Rule

``` text
Dedicated/high-capacity network
        |
        v
Use available

Shared production network
        |
        v
Set bandwidth limit
```

------------------------------------------------------------------------

# 9. Keep Deleted Files

Current value:

``` text
OFF
```

This is a very important setting.

## What?

It controls what happens when data exists in the destination but no
longer exists in the source.

With:

``` text
Keep deleted files = OFF
```

destination data that is not present in the source can be deleted during
the task.

------------------------------------------------------------------------

## Example

Before:

``` text
SOURCE                  DESTINATION

A.txt  ----------------> A.txt
B.txt  ----------------> B.txt
C.txt  ----------------> C.txt
```

Someone deletes:

``` text
SOURCE/C.txt
```

Now:

``` text
SOURCE                  DESTINATION

A.txt  ----------------> A.txt
B.txt  ----------------> B.txt
C.txt                   C.txt
                         ^
                         |
                    extra file
```

With:

``` text
Keep deleted files = OFF
```

DataSync can remove the destination copy:

``` text
SOURCE                  DESTINATION

A.txt  ----------------> A.txt
B.txt  ----------------> B.txt
C.txt                   C.txt deleted
```

------------------------------------------------------------------------

## Alternative: Keep deleted files = ON

Then:

``` text
SOURCE                  DESTINATION

A.txt  ----------------> A.txt
B.txt  ----------------> B.txt
C.txt                   C.txt remains
```

The destination copy is preserved even though the source copy is gone.

------------------------------------------------------------------------

## When to choose OFF?

Use:

``` text
Keep deleted = OFF
```

when the requirement is:

> **"Destination should mirror the source."**

------------------------------------------------------------------------

## When to choose ON?

Use:

``` text
Keep deleted = ON
```

when:

> **"Destination may contain files that must survive even if they
> disappear from the source."**

------------------------------------------------------------------------

## ⚠️ Important warning

Your current setting is:

``` text
Keep deleted files = OFF
```

Therefore, understand the consequence before scheduling the task.

If someone accidentally deletes source data:

``` text
Source deletion
      |
      v
DataSync runs
      |
      v
Destination copy can also be deleted
```

If the destination needs independent retention, this setting needs
careful consideration.

------------------------------------------------------------------------

# 10. Overwrite Files

Current value:

``` text
ON
```

## What?

If source data changes and the destination contains an older version,
DataSync can overwrite the destination version.

------------------------------------------------------------------------

## Example

Source:

``` text
config.txt = VERSION 2
```

Destination:

``` text
config.txt = VERSION 1
```

With:

``` text
Overwrite files = ON
```

after synchronization:

``` text
Destination:

config.txt = VERSION 2
```

------------------------------------------------------------------------

## Alternative: Overwrite OFF

If overwrite is disabled, existing destination files are not freely
replaced by the source version.

Example:

``` text
SOURCE                  DESTINATION

V2                       V1
 |                        |
 +------- sync ---------->|
                          |
                     remains V1
```

### When use OFF?

When the destination contains data that must be protected from being
overwritten.

### When use ON?

When the source is the authoritative copy and the destination should
follow it.

------------------------------------------------------------------------

# 11. Copy Ownership

Current value:

``` text
ON
```

## What?

Ownership includes information such as:

``` text
User ID
Group ID
```

For example on Linux:

``` text
appuser:appgroup
```

DataSync attempts to preserve ownership information where supported.

------------------------------------------------------------------------

## Example

Source:

``` text
-rw-r----- appuser appgroup config.yml
```

With ownership copying enabled, DataSync attempts to preserve the
owner/group information at the destination.

### Why?

Applications can depend on correct ownership.

For example:

``` text
Application
    |
    v
config.yml
    |
    v
Owner = appuser
```

If the ownership is wrong, the application may encounter permission
problems.

### Alternative

Disable ownership copying when destination ownership needs to be managed
independently.

------------------------------------------------------------------------

# 12. Copy Permissions

Current value:

``` text
ON
```

## What?

Preserves filesystem permissions where supported.

Example:

``` text
Source:

-rwxr-x---

script.sh
```

The destination attempts to retain the corresponding permissions.

------------------------------------------------------------------------

## Why?

Linux permissions control:

``` text
Read
Write
Execute
```

For example:

``` text
Application user
       |
       v
/application/config
       |
       v
Permission denied
```

A migration can fail operationally if important permissions are not
preserved.

### Alternative

Disable it when the destination has its own permission model and
permissions should be managed independently.

------------------------------------------------------------------------

# 13. Copy Timestamps

Current value:

``` text
ON
```

## What?

Preserves filesystem timestamps where supported, such as:

-   access time
-   modification time

------------------------------------------------------------------------

## Example

Source:

``` text
backup.sql
Modified = 2026-09-01 10:30
```

With timestamp copying enabled, DataSync attempts to retain the source
timestamp at the destination.

------------------------------------------------------------------------

## Why?

Some applications and scripts use timestamps to determine:

``` text
Was this file changed?
Is this file new?
Should this file be processed?
```

Timestamps are also useful when preserving the original state of a
filesystem during migration.

### Alternative

Disable it when destination timestamps should be generated/managed
independently.

------------------------------------------------------------------------

# 14. Queueing

Current value:

``` text
ON
```

## What?

Queueing allows a new execution to wait when a previous execution of the
task is still running.

This is about **task executions**, not simply queuing individual files.

------------------------------------------------------------------------

## Example

Execution 1:

``` text
DataSync Task
     |
     v
Execution #1
     |
     v
RUNNING
```

Another execution is started:

``` text
Execution #2
     |
     v
QUEUED
```

When execution #1 finishes:

``` text
Execution #1
     |
     v
COMPLETED
     |
     v
Execution #2
     |
     v
STARTS
```

### Why?

This is useful when:

-   tasks are scheduled
-   executions can overlap
-   a previous transfer may take longer than expected

### Alternative: Queueing OFF

A new execution won't simply wait in the task queue in the same way.
Depending on the situation, the new execution may not start because the
task is already running.

### Example

``` text
Execution #1 = RUNNING

Execution #2 starts
       |
       v
Cannot run concurrently
```

### Rule

``` text
Recurring/scheduled task
        |
        v
Queueing ON is often useful
```

------------------------------------------------------------------------

# 15. Schedule

Current value:

``` text
Not scheduled
```

## What?

The task does not automatically execute on a recurring schedule.

You can start it manually or through supported automation/API
mechanisms.

------------------------------------------------------------------------

## Alternative: Scheduled execution

You can configure the task to run periodically.

Conceptually:

``` text
09:00 -> DataSync
10:00 -> DataSync
11:00 -> DataSync
12:00 -> DataSync
```

Combined with:

``` text
Transfer only data that has changed
```

you get recurring synchronization:

``` text
SOURCE
   |
   v
DataSync
   |
   v
Only changed data
   |
   v
DESTINATION
```

### When use a schedule?

Use scheduling when the source and destination need regular
synchronization.

------------------------------------------------------------------------

# 16. Task Report

Current value:

``` text
None
```

## What?

A DataSync task report can provide detailed information about the
transfer, including details about files and the amount of data moved.

------------------------------------------------------------------------

## Alternative: Enable a task report

Useful information can include:

``` text
Files transferred
Files skipped
Files failed
Data transferred
Transfer results
```

### When useful?

Especially useful for:

-   migration projects
-   auditing
-   troubleshooting
-   compliance reporting
-   production operations

### Simple rule

``` text
Small/simple task
      |
      v
Report may be unnecessary

Important production migration
      |
      v
Detailed report can be valuable
```

------------------------------------------------------------------------

# 17. CloudWatch Logging

Current value:

``` text
Log level = Basic

Log group = /aws/datasync
```

## What?

DataSync can send task execution information to Amazon CloudWatch Logs.

------------------------------------------------------------------------

## Log level: OFF

No DataSync CloudWatch logging.

Use when detailed DataSync logs are not required.

------------------------------------------------------------------------

## Log level: BASIC

Provides basic information such as transfer errors.

Your current setting:

``` text
BASIC
```

Good for basic monitoring/troubleshooting.

------------------------------------------------------------------------

## Log level: TRANSFER

Provides more detailed transfer-level information.

This can be useful when you need more visibility into files/objects
being transferred and related integrity activity.

------------------------------------------------------------------------

## Example

``` text
DataSync Task
      |
      v
CloudWatch Logs
      |
      v
/aws/datasync
```

If the task encounters transfer errors, CloudWatch can help you
investigate what happened.

------------------------------------------------------------------------

# 18. How the Important Options Work Together

Your configuration is not just a collection of independent settings.

They work together.

Suppose:

``` text
SOURCE:

/data
├── file1.txt
├── file2.txt
├── file3.txt
└── old.txt
```

Destination:

``` text
/data
├── file1.txt
├── file2.txt
├── file3.txt
└── destination-only.txt
```

Now imagine:

``` text
file1.txt = unchanged
file2.txt = changed
file3.txt = unchanged

old.txt = deleted from source
destination-only.txt = exists only at destination
```

Your settings are:

``` text
Transfer mode = Changed
Keep deleted = OFF
Overwrite = ON
Verification = Transferred only
```

DataSync behaves conceptually like:

``` text
file1.txt
    |
    +--> unchanged
    |
    +--> skip transfer

file2.txt
    |
    +--> changed
    |
    +--> transfer
    |
    +--> overwrite destination
    |
    +--> verify

file3.txt
    |
    +--> unchanged
    |
    +--> skip transfer

old.txt
    |
    +--> no longer in source
    |
    +--> destination copy removed

destination-only.txt
    |
    +--> not present in source
    |
    +--> removed because Keep deleted = OFF
```

This is why your configuration behaves like a **source-authoritative
mirror**.

------------------------------------------------------------------------

# 19. Your Current Configuration in Plain English

Your task says:

> **Scan the entire source. Don't exclude anything. Compare source and
> destination and transfer only data/metadata that has changed. Verify
> the data that was transferred. Use available bandwidth. If something
> disappears from the source, remove its destination copy. If source
> data changes, overwrite the destination version. Preserve ownership,
> permissions, and timestamps. If another execution starts while one is
> running, queue it. Don't automatically schedule the task. Generate no
> task report and use basic CloudWatch logging.**

------------------------------------------------------------------------

# 20. Best Configuration by Scenario

## Scenario A --- Normal recurring synchronization

Recommended concept:

``` text
Contents to scan       = Everything
Transfer mode          = Changed
Verification           = Verify only transferred data
Bandwidth              = Use available / limit if WAN is shared
Keep deleted           = OFF if destination must mirror source
Overwrite              = ON
Ownership              = ON where applicable
Permissions            = ON where applicable
Timestamps             = ON
Queueing               = ON
Schedule               = Configure as required
Logging                = BASIC or TRANSFER depending on troubleshooting needs
```

------------------------------------------------------------------------

# 21. Scenario B --- Destination Must Never Lose Deleted Source Files

Use:

``` text
Keep deleted files = ON
```

Example:

``` text
Source:
A.txt
B.txt

Destination:
A.txt
B.txt
backup-old.txt
```

If `backup-old.txt` isn't in the source:

``` text
Keep deleted = ON
```

means:

``` text
backup-old.txt remains
```

------------------------------------------------------------------------

# 22. Scenario C --- Destination Must Exactly Follow Source

Use:

``` text
Transfer mode = Changed
Keep deleted = OFF
Overwrite = ON
```

Conceptually:

``` text
SOURCE
  |
  | authoritative
  v
DESTINATION
```

Source changes should flow to the destination.

Source deletions can also flow to the destination.

------------------------------------------------------------------------

# 23. Scenario D --- One-Time Full Copy

Use:

``` text
Transfer mode = Transfer all data
```

Concept:

``` text
SOURCE
  |
  | copy everything
  v
DESTINATION
```

This is different from an incremental synchronization task.

------------------------------------------------------------------------

# 24. Scenario E --- Shared Corporate Network

Suppose:

``` text
WAN = 1 Gbps

Users + Applications + DataSync
```

Don't automatically let DataSync consume everything.

Consider:

``` text
Bandwidth limit = configured limit
```

Example:

``` text
DataSync = 200 Mbps
```

This protects bandwidth for production traffic.

------------------------------------------------------------------------

# 25. Scenario F --- Critical Migration

For an important migration, consider stronger observability:

``` text
Transfer mode       = Changed / All depending on migration
Verification        = Verify all data when appropriate
Task report         = Enabled
Logging             = TRANSFER
```

The exact configuration should be based on the migration's requirements
and the destination storage type.

------------------------------------------------------------------------

# 26. Most Important Interview Questions

## Q1. What is DataSync?

**Answer:**

AWS DataSync is a managed service for transferring and synchronizing
data between supported storage systems.

------------------------------------------------------------------------

## Q2. What is the difference between Transfer all data and Transfer only data that has changed?

**Answer:**

`Transfer all data` copies all source content without using destination
comparison to identify changed data.

`Transfer only data that has changed` compares source and destination
and transfers only data/metadata that differs.

------------------------------------------------------------------------

## Q3. What happens when Keep deleted files is disabled?

**Answer:**

Data that exists in the destination but no longer exists in the source
can be deleted during the task, allowing the destination to mirror the
source.

------------------------------------------------------------------------

## Q4. What does Overwrite files do?

**Answer:**

It allows DataSync to replace existing destination data when the source
version or relevant metadata has changed.

------------------------------------------------------------------------

## Q5. Why copy permissions?

**Answer:**

To preserve filesystem permission behavior so applications and users
continue to have the expected read/write/execute access.

------------------------------------------------------------------------

## Q6. Why copy ownership?

**Answer:**

To preserve user/group ownership information where supported, which can
be important for applications and filesystem access.

------------------------------------------------------------------------

## Q7. What is the difference between Verify only transferred data and Verify all data?

**Answer:**

`Verify only transferred data` verifies the data transferred by the
task.

`Verify all data` verifies the entire source and destination to
determine whether they are fully synchronized.

------------------------------------------------------------------------

## Q8. What does Queueing do?

**Answer:**

It allows another task execution to wait when a previous execution is
still running instead of requiring concurrent execution.

------------------------------------------------------------------------

## Q9. What is the difference between DataSync and Storage Gateway?

``` text
DataSync
   =
Move/synchronize data

Storage Gateway
   =
Provide hybrid access to AWS-backed storage
```

------------------------------------------------------------------------

## Q10. What is the difference between DataSync and DMS?

``` text
DataSync
   =
File/object data transfer

DMS
   =
Database migration/replication
```

------------------------------------------------------------------------

# 27. Quick Decision Tree

``` text
Do I need to move/synchronize files or objects?
                |
               YES
                |
                v
           AWS DataSync
                |
                v
       What transfer behavior?
          /             \
         /               \
   Full copy          Recurring sync
      |                    |
      v                    v
Transfer all          Changed only
                           |
                           v
                 Should destination
                 mirror source?
                    /        \
                  YES         NO
                   |           |
                   v           v
              Keep deleted   Keep deleted
                   OFF          ON
                   |             |
                   v             v
               Overwrite      Consider
                  ON         destination
                             protection
```

------------------------------------------------------------------------

# 28. Final Cheat Sheet

  -----------------------------------------------------------------------
  Option                  Meaning                 If you choose the
                                                  alternative
  ----------------------- ----------------------- -----------------------
  **Everything**          Consider the complete   Specific paths only
                          source                  

  **Excludes**            Skip matching source    Nothing skipped if
                          data                    empty

  **Changed data**        Transfer differences    All data is copied

  **Verify transferred**  Verify transferred data None = no extra
                                                  post-transfer
                                                  verification; All =
                                                  verify entire dataset

  **Use available**       Don't impose a manual   Configured bandwidth
                          bandwidth cap           limit

  **Keep deleted OFF**    Destination-only data   ON = preserve
                          can be deleted          destination-only data

  **Overwrite ON**        Update destination with OFF = don't overwrite
                          changed source data     existing destination
                                                  data

  **Ownership ON**        Preserve ownership      Destination ownership
                                                  isn't copied

  **Permissions ON**      Preserve permissions    Destination permissions
                                                  aren't copied

  **Timestamps ON**       Preserve timestamps     Destination timestamps
                                                  can differ

  **Queueing ON**         Queue another execution No waiting queue
                          behind a running one    behavior

  **Schedule None**       No recurring automatic  Task runs periodically
                          schedule                

  **Report None**         No task report          Generate detailed
                                                  transfer report

  **Basic logging**       Basic CloudWatch        OFF or more detailed
                          information/errors      TRANSFER logging
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 29. One-Line Memory Trick

Remember DataSync options in this order:

``` text
SCAN
  ↓
EXCLUDE
  ↓
COMPARE
  ↓
TRANSFER
  ↓
VERIFY
  ↓
DESTINATION HANDLING
  ↓
METADATA
  ↓
QUEUE
  ↓
SCHEDULE
  ↓
REPORT
  ↓
LOGGING
```

Or simply:

> **What to copy → What changed → Verify → What happens to destination →
> Preserve metadata → Run/monitor the task.**

------------------------------------------------------------------------

# 30. Your Current Configuration --- Final Verdict

Your current settings are logically consistent for a
**source-authoritative synchronization task**:

``` text
Everything
    +
No exclusions
    +
Changed data only
    +
Verify transferred data
    +
Use available bandwidth
    +
Delete destination data missing from source
    +
Overwrite changed destination files
    +
Preserve ownership
    +
Preserve permissions
    +
Preserve timestamps
    +
Queue overlapping executions
```

The biggest things to be careful about are:

### ⚠️ 1. Keep deleted files = OFF

Source deletion can propagate to the destination.

### ⚠️ 2. Overwrite = ON

Source changes can overwrite destination versions.

### ⚠️ 3. Use available bandwidth

DataSync may consume significant available network capacity.

### ⚠️ 4. Schedule

Your task currently says **Not scheduled**, so it is not automatically
running on a recurring schedule.

### ⚠️ 5. Logging/reporting

You currently have **no task report** and **Basic CloudWatch logging**.
For important production migrations, stronger reporting/logging may be
useful.

------------------------------------------------------------------------

## Final mental model

``` text
             AWS DATASYNC TASK

SOURCE
  |
  |-- What to scan?
  |      └── Everything
  |
  |-- What to exclude?
  |      └── Nothing
  |
  |-- What to transfer?
  |      └── Changed data
  |
  |-- How to verify?
  |      └── Transferred data
  |
  |-- How much bandwidth?
  |      └── Available
  |
  |-- What about deleted files?
  |      └── Delete from destination
  |
  |-- What about changed files?
  |      └── Overwrite destination
  |
  |-- Preserve metadata?
  |      ├── Ownership ✓
  |      ├── Permissions ✓
  |      └── Timestamps ✓
  |
  |-- What if another execution starts?
  |      └── Queue it
  |
  |-- Automatic schedule?
  |      └── No
  |
  |-- Detailed report?
  |      └── No
  |
  └-- CloudWatch?
         └── Basic
                |
                v
           DESTINATION
```

**Bottom line:** your current configuration is essentially saying
**"Source is the master; keep the destination synchronized with it,
transfer only differences, preserve filesystem metadata, and propagate
source deletions/updates."**
