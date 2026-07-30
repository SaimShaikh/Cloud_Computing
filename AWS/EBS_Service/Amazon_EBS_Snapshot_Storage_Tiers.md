# Amazon EBS Snapshot Storage Tiers

## Overview

Amazon **EBS Snapshots** support **two snapshot storage tiers**:

1.  **Standard Snapshot Tier**
2.  **Archive Snapshot Tier**

> **Note:** These are **snapshot storage tiers**, **not** EBS volume
> types.

------------------------------------------------------------------------

# 1. Standard Snapshot Tier

The **Standard Snapshot Tier** is the default storage tier for every new
EBS snapshot.

### Characteristics

-   Immediate access
-   Fast recovery
-   Can directly create an EBS volume
-   Suitable for operational backups
-   Higher storage cost than Archive

### Best Use Cases

-   Daily backups
-   Weekly backups
-   Disaster recovery
-   Frequently accessed snapshots
-   Test and development restores

### Workflow

``` text
EC2 Instance
      │
      ▼
EBS Volume
      │
Create Snapshot
      ▼
Standard Snapshot Tier
      │
Create Volume
      ▼
Attach to EC2
```

------------------------------------------------------------------------

# 2. Archive Snapshot Tier

The **Archive Snapshot Tier** is a lower-cost storage tier designed for
**long-term retention**.

Snapshots stored in the Archive tier **cannot be used immediately**.

They must first be **restored to the Standard tier** before creating an
EBS volume.

### Characteristics

-   Lower storage cost
-   Long-term retention
-   Compliance and audit backups
-   Restore required before use
-   Retrieval takes hours

### Best Use Cases

-   Compliance
-   Audit requirements
-   Monthly backups
-   Yearly backups
-   Long-term disaster recovery

### Workflow

``` text
Standard Snapshot
        │
Move to Archive
        ▼
Archive Snapshot
        │
Restore
        ▼
Standard Snapshot
        │
Create Volume
        ▼
EC2 Instance
```

------------------------------------------------------------------------

# Standard vs Archive

  Feature          Standard Snapshot Tier   Archive Snapshot Tier
  ---------------- ------------------------ -----------------------------
  Purpose          Operational backups      Long-term retention
  Access           Immediate                Restore required
  Recovery Speed   Fast                     Hours after restore request
  Cost             Higher                   Lower
  Best For         Daily/Weekly backups     Compliance & archival

------------------------------------------------------------------------

# Important Difference

## Amazon EBS Volume Types

These are the storage volumes attached to EC2 instances.

-   gp3
-   gp2
-   io2
-   io1
-   st1
-   sc1

These are called **Amazon EBS Volume Types**.

------------------------------------------------------------------------

## Amazon EBS Snapshot Storage Tiers

These are the storage tiers for snapshots.

-   Standard Snapshot Tier
-   Archive Snapshot Tier

These are called **Amazon EBS Snapshot Storage Tiers**.

------------------------------------------------------------------------

# Cost Optimization Example

A common lifecycle policy is:

``` text
Daily Snapshot
        │
Keep for 30 Days
        │
Move to Archive
        │
Keep for 1 Year
        │
Delete
```

Benefits:

-   Reduced storage costs
-   Automated retention
-   Compliance support
-   Long-term backup management

------------------------------------------------------------------------

# Interview Question

**Q: How many storage tiers are available for Amazon EBS Snapshots?**

**Answer:**

Amazon EBS Snapshots provide **two storage tiers**:

1.  **Standard Snapshot Tier** -- Used for active snapshots that require
    fast recovery.
2.  **Archive Snapshot Tier** -- A lower-cost storage tier for long-term
    retention. Archived snapshots must be restored to the Standard tier
    before they can be used to create an EBS volume.

------------------------------------------------------------------------

# Key Takeaways

-   Amazon EBS Snapshots have **2 storage tiers**.
-   Every new snapshot is created in the **Standard Snapshot Tier**.
-   Older snapshots can be moved to the **Archive Snapshot Tier** to
    reduce storage costs.
-   Archived snapshots must be restored before they can be used.
-   Snapshot storage tiers are different from EBS volume types.
