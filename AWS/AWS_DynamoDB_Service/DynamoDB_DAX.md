# DynamoDB Accelerator (DAX) – The Ultimate Guide

> Speed up Amazon DynamoDB reads to **single‑digit milliseconds** (and often microseconds) with a **managed, in‑memory cache** that’s API‑compatible with DynamoDB.

---

## 🧭 TL;DR

* **What:** Fully managed, **write‑through/read‑through** caching layer for DynamoDB.
* **Why:** Reduce read latency and offload read traffic from DynamoDB tables.
* **How:** Drop‑in replacement client (DAX SDK) that speaks DynamoDB APIs; cache sits in your **VPC**.
* **When:** Read‑heavy, bursty workloads, hot partitions/keys, leaderboard/feed/timeline/real‑time dashboards.
* **Gotchas:**

  * DAX accelerates **eventually consistent** reads; strongly consistent reads and transactions may bypass cache.
  * **VPC‑only**, private endpoints; you **must** use a DAX‑aware client.
  * Cache **isn’t a database**—no backups/restore; treat as ephemeral.

---

## 📁 Repo Structure

```
.
├── README.md                 # You are here
├── examples/
│   ├── nodejs/               # DAX with Node.js (amazon-dax-client)
│   ├── python/               # DAX with Python (amazondax)
│   └── java/                 # DAX with Java (AmazonDaxClient)
├── diagrams/
│   ├── dax-arch.png          # Architecture diagram (add your export)
│   └── dax-flow.png          # Request flow diagram
├── ops/
│   ├── cloudwatch-metrics.md # Key metrics & alarms
│   └── runbook.md            # Troubleshooting & ops playbook
└── LICENSE
```

> Tip: Keep **diagrams** and **runbooks** next to code. Caching bugs love vague docs.

---

## 1) What is DAX?

Amazon DynamoDB Accelerator (DAX) is a **fully managed in‑memory cache** for DynamoDB. It lives inside your **Amazon VPC**, runs as a **multi‑AZ cluster**, and exposes a client endpoint you use **instead of** your normal DynamoDB client. DAX understands DynamoDB requests (e.g., `GetItem`, `Query`, `Scan`, `BatchGetItem`) and returns cached responses when possible.

**Core traits**

* **API‑aware cache** (not a generic key/value): understands DynamoDB request/response shapes.
* **Write‑through**: writes go to DynamoDB and the cache is **updated/invalidated** automatically.
* **Managed HA**: multi‑AZ clusters with one primary and read replicas.
* **Microsecond‑to‑millisecond** read latency on cache hit.

---

## 2) Why use DAX? (Benefits)

* **Ultra‑low read latency** for hot keys and query patterns.
* **Offload read traffic** from DynamoDB ⇒ lower RCUs, smoother throttling profile.
* **Transparent caching**: minimal app changes vs rolling your own Redis + invalidation.
* **Managed service**: patching, failure handling, node replacement are AWS‑managed.
* **VPC isolation & encryption**: at‑rest and in‑transit encryption, SG/IAM controls.

---

## 3) When NOT to use DAX

* You need **strictly strongly consistent reads** on the fast path.
* **Write‑heavy** workloads where cache hit rate will be low.
* **Large, non‑repeatable scans** with poor locality (low reuse).
* You require a **generic cache** for other services (consider ElastiCache/Redis/Memcached).
* **Cross‑VPC/public** access needed (DAX is VPC‑only; use VPC peering/Transit Gateway if necessary).

---

## 4) Architecture (How DAX works)

**Cluster layout**

* **Primary node**: coordinates write‑through and cluster membership.
* **Read replicas (1–9)**: serve cached reads; multi‑AZ for HA.
* **Client endpoint**: your app connects to the **DAX cluster endpoint** via the DAX client SDK.

**Request flow**

1. App calls DynamoDB API **via DAX client**.
2. **Cache hit** → DAX returns from memory.
3. **Cache miss** → DAX fetches from DynamoDB, **stores** item/query result, returns to the app.
4. **Write** (`PutItem`, `UpdateItem`, `DeleteItem`) → sent to DynamoDB and **cache updated/invalidated**.

**Consistency**

* DAX speeds up **eventually consistent** reads.
* **Strongly consistent reads** and **transactions** are typically **routed to DynamoDB** by the client (no cache hit). Use only when you truly need them.

---

## 5) Setup Guide (Step‑by‑Step)

### Prereqs

* Existing **DynamoDB table(s)** in the same account/region.
* **VPC** with **subnets** in at least **2 AZs**; suitable **security groups**.
* App language with a **DAX‑aware client** (Java, Node.js, Python, .NET).

### Create subnet group & cluster

1. **DAX Subnet Group**: pick private subnets in ≥2 AZs.
2. **Security group**: allow inbound from your app instances/ENIs; restrict egress as needed.
3. **Create DAX Cluster**: choose **node type**, **replica count**, **encryption**, parameter group.
4. **Grab the cluster endpoint** (e.g., `my-dax.abc123.clustercfg.dax.use1.cache.amazonaws.com:8111`).

### Wire up the client

* **Replace** standard DynamoDB client with the **DAX client** using the cluster endpoint.
* Keep your **DynamoDB client** around for ops paths that require strong consistency/transactions.

> ⚠️ Don’t point non‑DAX SDKs to the DAX endpoint; they won’t understand it.

---

## 6) Code Examples

### Node.js

```bash
npm i amazon-dax-client aws-sdk
```

```js
// examples/nodejs/index.js
const AWS = require('aws-sdk');
const AmazonDaxClient = require('amazon-dax-client');

const dax = new AmazonDaxClient({
  endpoints: ['my-dax.abc123.clustercfg.dax.us-east-1.cache.amazonaws.com:8111'],
  region: 'us-east-1',
});

const dynamodb = new AWS.DynamoDB({ service: dax });
const docClient = new AWS.DynamoDB.DocumentClient({ service: dynamodb });

async function main() {
  const params = { TableName: 'Users', Key: { userId: 'u_123' } };
  const res = await docClient.get(params).promise();
  console.log('Item:', res.Item);
}

main().catch(console.error);
```

### Python

```bash
pip install amazondax boto3
```

```python
# examples/python/app.py
from amazondax import AmazonDaxClient
import boto3

region = 'us-east-1'
dax_endpoint = 'my-dax.abc123.clustercfg.dax.us-east-1.cache.amazonaws.com:8111'

# DAX-aware low-level client
dax = AmazonDaxClient(region_name=region, endpoints=[dax_endpoint])

# Wrap with DynamoDB resource via botocore adapter
import botocore.session
session = botocore.session.get_session()
session.register_component('dynamodb', dax)

dynamodb = boto3.resource('dynamodb', region_name=region)

table = dynamodb.Table('Users')
item = table.get_item(Key={'userId': 'u_123'}).get('Item')
print(item)
```

### Java

```xml
<!-- pom.xml -->
<dependency>
  <groupId>com.amazonaws</groupId>
  <artifactId>amazon-dax-client</artifactId>
  <version>latest</version>
</dependency>
```

```java
// examples/java/App.java
import com.amazon.dax.client.dynamodbv2.AmazonDaxClient;
import com.amazonaws.services.dynamodbv2.AmazonDynamoDB;
import com.amazonaws.services.dynamodbv2.document.DynamoDB;
import com.amazonaws.services.dynamodbv2.document.Table;
import com.amazonaws.services.dynamodbv2.document.Item;

public class App {
  public static void main(String[] args) throws Exception {
    String region = "us-east-1";
    String endpoint = "my-dax.abc123.clustercfg.dax.us-east-1.cache.amazonaws.com:8111";

    AmazonDynamoDB dax = AmazonDaxClient.builder()
      .withRegion(region)
      .withEndpointConfiguration(endpoint)
      .build();

    DynamoDB db = new DynamoDB(dax);
    Table table = db.getTable("Users");
    Item item = table.getItem("userId", "u_123");
    System.out.println(item.toJSONPretty());
  }
}
```

> **Note:** SDK snippets may require minor glue depending on library versions. Pin versions in production.

---

## 7) Sizing & Performance Tuning

* **Right‑size nodes**: start small, observe **CPUUtilization**, **Evictions**, **CacheHitRate**.
* **Replica count**: scale **reads** horizontally; aim for multi‑AZ.
* **Hot keys**: denormalize, shard hot items, or pre‑compute aggregates to avoid single‑key hotspots.
* **TTL/Item size**: smaller items cache better; use projection expressions to fetch only needed attributes.
* **Query shape**: prefer **Query** over **Scan**; design keys for locality & reuse.

---

## 8) Security & Networking

* **VPC‑only** service with **ENIs**; no public internet.
* **IAM** SigV4 auth via the DAX client.
* **Encryption** at rest (KMS) and in transit (TLS to client).
* **Security Groups** to restrict who can talk to the cluster.

---

## 9) Operations: Metrics, Alarms, and Scaling

Key **CloudWatch metrics** to watch:

* `CacheHitRate` (target high; watch per‑op types)
* `Evictions` (if rising, cache pressure)
* `RequestCount`, `ThrottledRequests`, `Errors`
* `CPUUtilization`, `FreeableMemory`, `Network*`

**Alarms**

* Low `CacheHitRate` for sustained periods
* High `Evictions` or `CPUUtilization`
* Any `ThrottledRequests` > 0 after warmup

**Scaling**

* **Vertical**: change node type.
* **Horizontal**: add replicas (zero‑downtime) for read scale & HA.

**Maintenance**

* Maintenance windows handled by AWS; cluster remains available via replicas.

---

## 10) Pricing Model (High Level)

* **Per‑node hourly** price (varies by **region** and **node type**).
* **Data transfer** within the same AZ is typically included; cross‑AZ may incur charges.
* You pay for **provisioned nodes**, not per‑request. For exact numbers, check the official **AWS DAX Pricing** page for your region.

> Rule of thumb: If DAX meaningfully reduces DynamoDB RCUs and keeps hit rate high, it can pay for itself on hot, read‑heavy workloads.

---

## 11) Limits & Quotas (Common)

* **Nodes per cluster**: up to **10** (1 primary + up to 9 replicas).
* **Item size**: subject to DynamoDB limits; very large items reduce cache efficiency.
* **API coverage**: Most common ops cached; strong reads/transactions may bypass.
* **Region/VPC** restrictions: Cluster and clients must be in compatible networking.

(Always verify current limits in AWS docs; service quotas evolve.)

---

## 12) Patterns & Best Practices

* **Write‑through + TTL**: let DAX manage invalidation; set reasonable TTLs for freshness.
* **Dual‑client strategy**: DAX for fast path; native DynamoDB client for ops needing strong consistency/transactions.
* **Warmup** critical keys on deploys to avoid cold starts.
* **Backpressure**: keep native DynamoDB fallbacks for cluster failover scenarios.
* **Observability first**: log cache hit/miss rates at call sites during ramp‑up.

---

## 13) DAX vs Alternatives

| Feature      | **DAX**                                                     | **ElastiCache (Redis/Memcached)**      | **"No Cache" (Raw DynamoDB)** |
| ------------ | ----------------------------------------------------------- | -------------------------------------- | ----------------------------- |
| Type         | Managed, **API‑aware** cache for DynamoDB                   | Generic in‑memory cache                | Fully managed NoSQL DB        |
| Integration  | **Drop‑in** DAX client (DynamoDB APIs)                      | App‑managed keys & invalidation        | N/A                           |
| Read Latency | **µs–ms (hit)**                                             | µs–ms (hit)                            | 1–10ms typical                |
| Consistency  | **Eventually consistent** reads cached; strong reads bypass | You define semantics                   | Strong/eventual per request   |
| Writes       | **Write‑through** via DAX client                            | App logic                              | Direct                        |
| HA           | Multi‑AZ cluster                                            | Redis/Memcached architectures vary     | Regional, multi‑AZ table      |
| Use Cases    | DynamoDB‑centric, read‑heavy                                | Cross‑service caching, custom patterns | Simpler stacks, lower reuse   |

---

## 14) Troubleshooting & Runbook

* **Miss rate high**: Confirm you’re using **Query** over **Scan**, check hot keys, verify TTL, ensure you’re actually calling via the **DAX client**.
* **Throttles**: Inspect DynamoDB table capacity, DAX CPU/memory; scale replicas.
* **Consistent reads slow**: They bypass cache—use only where necessary or revisit consistency requirements.
* **Connectivity**: Security group rules, route tables, DNS in VPC; ensure client is in **same region/VPC** path.
* **Serialization issues**: Align SDK versions; test with a minimal repro.

---

## 15) FAQs

**Q: Do I need to change my data model?**
A: Usually no. Good partition key design still matters for cache locality.

**Q: Does DAX cache `Scan`?**
A: It can, but scans are often low‑reuse. Prefer `Query` with key‑based access.

**Q: What happens on node failure?**
A: Cluster fails over within the AZ set; you keep using the same endpoint.

**Q: Is DAX a replacement for Redis?**
A: Not generally. DAX is DynamoDB‑aware. Redis is great for cross‑service/generic caching.

---

## 16) Local Dev & Testing

* Use **feature flags** to toggle DAX on/off in non‑prod.
* Provide **fall‑back** path to native DynamoDB client for parity checks.
* Add **integration tests** that assert expected hit/miss behavior on hot keys.

---

