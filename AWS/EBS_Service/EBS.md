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

| Volume Type                        | Definition                                                                                                                                | Storage Type | Primary Use Cases                                                                                                              | Performance Characteristics                                                                                             | Benefits                                                                                                        | Limitations                                                                                                                                               | Boot Volume Support | Recommended For                                |
| ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------- | ---------------------------------------------- |
| **gp3 (General Purpose SSD)**      | AWS's latest generation SSD volume that provides independent scaling of storage, IOPS, and throughput. Designed for most workloads.       | SSD          | Web Servers, Application Servers, Kubernetes Nodes, Jenkins, Docker Hosts, Small to Medium Databases, Production EC2 Workloads | **3,000 IOPS** and **125 MB/s** included. Scales up to **16,000 IOPS** and **1,000 MB/s** independently of volume size. | Lower cost than gp2, predictable performance, no burst credits, flexible scaling, best price/performance ratio. | Cannot reach extremely high IOPS like io2. No Multi-Attach support.                                                                                       | ✅ Yes               | Default choice for almost all new deployments. |
| **gp2 (General Purpose SSD)**      | Previous generation SSD volume where performance is tied directly to volume size.                                                         | SSD          | Legacy applications, existing deployments using gp2.                                                                           | Provides **3 IOPS per GiB**. Small volumes rely on burst credits. Maximum 16,000 IOPS.                                  | Easy to use, reliable, widely supported.                                                                        | Must increase storage size to gain more IOPS. Burst credits can be exhausted, causing performance drops. More expensive than gp3 for similar performance. | ✅ Yes               | Existing workloads not yet migrated to gp3.    |
| **io2 (Provisioned IOPS SSD)**     | Enterprise-grade SSD volume designed for mission-critical applications requiring extremely high performance, low latency, and durability. | SSD          | Oracle Database, SAP HANA, Microsoft SQL Server, PostgreSQL, Financial Applications, Enterprise Databases                      | Up to **256,000 IOPS**, **4,000 MB/s throughput**, sub-millisecond latency, 99.999% durability.                         | Highest performance, consistent latency, supports Multi-Attach, highest durability.                             | Most expensive EBS volume type. Overkill for general workloads.                                                                                           | ✅ Yes               | Mission-critical production databases.         |
| **io1 (Provisioned IOPS SSD)**     | Older provisioned IOPS SSD designed for applications requiring consistent high performance.                                               | SSD          | Legacy enterprise databases and workloads requiring fixed IOPS.                                                                | Up to **64,000 IOPS** and **1,000 MB/s throughput**.                                                                    | Consistent IOPS performance, suitable for database workloads.                                                   | Lower durability than io2, no Multi-Attach, generally replaced by io2.                                                                                    | ✅ Yes               | Older workloads not yet migrated to io2.       |
| **st1 (Throughput Optimized HDD)** | Low-cost HDD volume optimized for high-throughput sequential workloads rather than random I/O.                                            | HDD          | Hadoop Clusters, Data Warehouses, Log Processing, Streaming Analytics, ETL Jobs, Big Data Applications                         | Up to **500 MB/s throughput**, optimized for large sequential reads/writes.                                             | Cost-effective for large datasets, excellent throughput performance.                                            | Poor random I/O performance, not suitable for databases or boot volumes.                                                                                  | ❌ No                | Big Data and analytics workloads.              |
| **sc1 (Cold HDD)**                 | Lowest-cost EBS volume designed for infrequently accessed data.                                                                           | HDD          | Archives, Compliance Data, Historical Logs, Backup Repositories, Cold Storage                                                  | Up to **250 MB/s throughput**, optimized for infrequent access.                                                         | Lowest storage cost among EBS volumes.                                                                          | Lowest performance, unsuitable for active workloads, poor random access performance.                                                                      | ❌ No                | Long-term archival storage.                    |


## SSD vs HDD Volume Family

| Feature                | SSD Volumes (gp3, gp2, io2, io1)    | HDD Volumes (st1, sc1) |
| ---------------------- | ----------------------------------- | ---------------------- |
| Access Pattern         | Random + Sequential                 | Sequential             |
| Latency                | Low (milliseconds/sub-milliseconds) | Higher                 |
| Database Support       | ✅ Excellent                         | ❌ Not Recommended      |
| Boot Volume Support    | ✅ Yes                               | ❌ No                   |
| Transaction Processing | ✅ Excellent                         | ❌ Poor                 |
| Cost                   | Higher                              | Lower                  |
| Big Data Analytics     | ⚠️ Possible                         | ✅ Excellent            |
| Archive Storage        | ❌ Costly                            | ✅ Best Choice          |


---

### How EBS Works:
1. Create an EBS volume in the same **region** and **AZ** as your EC2 instance.
2. Attach it to the EC2 instance.
3. Format and mount it to a directory.
4. Use it like any local disk.

---

**Example EBS Lifecycle:**
1. **Create** → 2. **Attach** → 3. **Format & Mount** → 4. **Use** → 5. **Backup (Snapshot)** → 6. **Detach/Delete**
