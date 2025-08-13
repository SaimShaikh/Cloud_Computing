<img width="500" height="282" alt="image" src="https://github.com/user-attachments/assets/9ff9c371-42bb-4330-b6d4-b3cd3aa3bf05" />

# 🧱 Placement Groups in AWS

A **Placement Group** in AWS is a logical grouping of EC2 instances to influence the placement strategy for performance, availability, or fault tolerance.

---

## 📌 Types of Placement Groups

### 1. Cluster Placement Group
- **Purpose**: Achieve low-latency, high-throughput networking.
- **Behavior**: Instances are placed close together in the same Availability Zone.
- **Use Case**: High Performance Computing (HPC), Big Data, Machine Learning workloads.
- **Limitation**: All instances must launch together, or the request may fail.

---

### 2. Spread Placement Group
- **Purpose**: Maximize availability by spreading instances across distinct hardware.
- **Behavior**: Each instance is placed on different racks or infrastructure.
- **Use Case**: Critical workloads needing instance isolation (e.g., domain controllers).
- **Limit**: 7 running instances per Availability Zone per group (can be increased via request).

---

### 3. Partition Placement Group
- **Purpose**: Distribute instances across partitions with fault isolation.
- **Behavior**:
  - Each partition has its own rack (network, power, hardware).
  - Failure in one partition doesn’t affect others.
- **Use Case**: Large-scale distributed systems (e.g., HDFS, Kafka, Cassandra).
- **Default Partition Limit**: 7 partitions per AZ.

---

## 🧾 Summary Table

| Placement Group Type | Goal                          | AZ Scope     | Isolation Level           | Common Use Case               |
|----------------------|-------------------------------|--------------|----------------------------|-------------------------------|
| Cluster              | Low latency, high throughput  | Single AZ    | Minimal (same rack)        | HPC, ML, Big Data             |
| Spread               | High availability             | Multi-AZ     | Maximum (across hardware)  | Critical isolated workloads   |
| Partition            | Balanced availability & scale | Multi-AZ     | Per partition (separate HW)| Distributed storage systems   |

---

> ✅ **Note**: Placement groups only affect EC2 instance placement. They do not influence pricing or automatic failover behavior.

