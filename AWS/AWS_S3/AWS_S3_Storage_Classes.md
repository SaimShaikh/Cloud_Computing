# AWS S3 Storage Classes

Amazon S3 provides multiple storage classes designed for different use cases, balancing **cost**, **availability**, **performance**, and **durability**. All S3 storage classes provide **99.999999999% (11 nines) durability**.

---

## **1. S3 Standard**
**Definition:**  
A general-purpose storage class for frequently accessed data with low latency and high throughput.

**Key Features:**
- High durability and availability (99.99%)
- Millisecond access latency
- Data is stored redundantly across multiple Availability Zones (AZs)

**Use Cases:**
- Static website hosting
- Frequently accessed application data
- Content delivery

---

## **2. S3 Intelligent-Tiering**
**Definition:**  
Automatically moves data between frequent and infrequent access tiers based on usage patterns to optimize costs.

**Key Features:**
- Automatic cost optimization
- No retrieval charges
- Ideal for data with unpredictable access patterns

**Use Cases:**
- Data lakes
- User-generated content
- Logs or analytics data with varying access frequency

---

## **3. S3 Standard-IA (Infrequent Access)**
**Definition:**  
A lower-cost storage option for data that is infrequently accessed but needs quick access when required.

**Key Features:**
- Lower cost than S3 Standard
- Retrieval fees apply
- 99.9% availability

**Use Cases:**
- Backup and recovery
- Disaster recovery
- Infrequently used but important files

---

## **4. S3 One Zone-IA**
**Definition:**  
Stores data in a **single Availability Zone**, offering lower cost but reduced resilience compared to multi-AZ classes.

**Key Features:**
- Lower storage cost
- Suitable for non-critical or easily re-creatable data
- 99.5% availability

**Use Cases:**
- Secondary backups
- Data that can be easily regenerated

---

## **5. S3 Glacier Instant Retrieval**
**Definition:**  
Low-cost storage class for data that is **rarely accessed but needs to be retrieved instantly**.

**Key Features:**
- Instant retrieval
- Low storage cost
- Ideal for archives requiring immediate access

**Use Cases:**
- Medical records
- Media archives
- Compliance documents

---

## **6. S3 Glacier Flexible Retrieval (formerly S3 Glacier)**
**Definition:**  
Low-cost archival storage for data that can tolerate retrieval times of minutes to hours.

**Key Features:**
- Very low cost
- Flexible retrieval options (Expedited, Standard, Bulk)
- Data retrieval time from minutes to hours

**Use Cases:**
- Long-term backups
- Disaster recovery
- Historical data storage

---

## **7. S3 Glacier Deep Archive**
**Definition:**  
The **lowest-cost storage** for long-term data retention, with retrieval times of up to 12 hours.

**Key Features:**
- Cheapest storage class
- Retrieval time: 12–48 hours
- 99.9% availability

**Use Cases:**
- Regulatory compliance archives
- Long-term backups
- Rarely accessed but mandatory data

---

## **8. S3 Reduced Redundancy Storage (RRS)** *(Legacy)*
**Definition:**  
Used to provide lower-cost storage by storing data with less redundancy.  
**Note:** Not recommended anymore; mostly replaced by One Zone-IA.

**Use Cases:**
- Non-critical, easily reproducible data
- Temporary data storage

---

## **Comparison Table**

| Storage Class | Availability | Durability | Retrieval | Cost |
|---------------|-------------|------------|-----------|-------|
| **Standard** | 99.99% | 11 nines | Instant | High |
| **Intelligent-Tiering** | 99.9%-99.99% | 11 nines | Instant | Optimized |
| **Standard-IA** | 99.9% | 11 nines | Instant (fee) | Lower |
| **One Zone-IA** | 99.5% | 11 nines | Instant (fee) | Lower |
| **Glacier Instant Retrieval** | 99.9% | 11 nines | Instant | Low |
| **Glacier Flexible Retrieval** | 99.99% | 11 nines | Minutes-Hours | Very Low |
| **Glacier Deep Archive** | 99.99% | 11 nines | Hours (12-48h) | Lowest |

---

## **Key Notes**
- All storage classes ensure **11 nines of durability**.
- Intelligent-Tiering is great for unpredictable workloads.
- Glacier tiers are best for cost savings with archival data.

---

## **References**
- [AWS S3 Documentation](https://docs.aws.amazon.com/AmazonS3/latest/dev/storage-class-intro.html)
