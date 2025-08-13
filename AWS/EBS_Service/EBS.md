# What is EBS?
**EBS (Elastic Block Store)** is a **block storage service** provided by AWS that is used with **EC2 instances**.  
It works like a virtual hard drive for your cloud server — you can attach, detach, resize, and back it up independently of the instance.

### Key Features:
- **Persistent Storage** – Data remains even after you stop or restart the EC2 instance (unless you delete the volume).
- **Block-Level Storage** – Works like a traditional hard disk, allowing random read/write access.
- **Scalable** – You can increase size or change performance type without losing data.
- **Availability** – Tied to a single **Availability Zone** (AZ) but can be moved via snapshots.
- **Backup via Snapshots** – You can create point-in-time backups stored in Amazon S3.

---

### EBS Volume Types:
| Volume Type   | Description | Best For |
|---------------|-------------|----------|
| **gp3**       | General-purpose SSD | Most workloads, good balance of price & performance |
| **gp2**       | Older general-purpose SSD | Legacy workloads |
| **io2 / io1** | Provisioned IOPS SSD | High-performance databases |
| **st1**       | Throughput-optimized HDD | Big data, streaming workloads |
| **sc1**       | Cold HDD | Infrequently accessed data |

---

### How EBS Works:
1. Create an EBS volume in the same **region** and **AZ** as your EC2 instance.
2. Attach it to the EC2 instance.
3. Format and mount it to a directory.
4. Use it like any local disk.

---

**Example EBS Lifecycle:**
1. **Create** → 2. **Attach** → 3. **Format & Mount** → 4. **Use** → 5. **Backup (Snapshot)** → 6. **Detach/Delete**
