# Amazon S3 Pricing 

> **Last verified:** August 29, 2025 · **Region shown in examples:** US East (N. Virginia) `us-east-1`
>
> Pricing varies by Region and can change. Always cross-check with the **[AWS S3 Pricing page](https://aws.amazon.com/s3/pricing/)** and the **[AWS Pricing Calculator](https://calculator.aws/#/createCalculator/S3)** before committing.

---

## 📌 What you pay for (at a glance)

1. **Storage (\$/GB-month)** — depends on **storage class** and **usage tier**.
2. **Requests (\$ per 1,000 requests)** — PUT/COPY/POST/LIST, GET, SELECT, etc.
3. **Data transfer out (\$/GB)** — internet egress, inter-Region, etc. (ingress is free).
4. **Features** — e.g., **Intelligent-Tiering monitoring**, **Replication**, **Object Lambda**, **S3 Tables**, etc.

---

## 🚀 Quick Start: Typical Baseline Prices (us-east-1)

These are commonly referenced public prices for `us-east-1`. Use them as **starting points** for estimates.

### Storage (GB-month)

* **S3 Standard** (frequent access):

  * First 50 TB / month: **\$0.023 per GB**
  * Next 450 TB / month: **\$0.022 per GB**
  * Over 500 TB / month: **\$0.021 per GB**
* **S3 Standard-IA** (infrequent access): *\~\$0.0125 per GB* (30‑day min, 128 KB min billable object)
* **S3 One Zone-IA**: *\~\$0.01 per GB* (single AZ, 30‑day min, 128 KB min billable object)
* **S3 Glacier Instant Retrieval**: *\~\$0.004 per GB* (90‑day min, 128 KB min billable object)
* **S3 Glacier Flexible Retrieval** (formerly Glacier): *Archive pricing; 90‑day min*
* **S3 Glacier Deep Archive**: **\$0.00099 per GB** (180‑day min)

> **Minimum storage duration / minimum billable object size** apply to many archive/IA classes. Intelligent‑Tiering charges a small **per‑object monitoring fee** and has **no retrieval charges**.

### Requests (per 1,000 requests)

* **PUT/COPY/POST/LIST**: **\$0.005**
* **GET and all other retrievals**: **\$0.0004**
* **Lifecycle transition** requests: **\$0.01** (to the target class; some classes differ)
* **DELETE/CANCEL**: **Free**

> Console browsing also generates request charges (e.g., LIST/GET) the same as API/SDK.

### Free Tier (new AWS accounts)

* **5 GB** S3 Standard storage
* **20,000 GET** requests
* **2,000 PUT/COPY/POST/LIST** requests
* **100 GB** data transfer out per month (not in GovCloud)

> From **July 15, 2025**, AWS also offers **up to \$200 Free Tier credits** for new customers (terms apply).

---

## 🧠 Storage Classes — When to use what

| Class                          | Typical use                           | Key notes                                                        |
| ------------------------------ | ------------------------------------- | ---------------------------------------------------------------- |
| **Standard**                   | Hot data, websites, analytics, ML     | Highest availability, tiered \$/GB-month                         |
| **Intelligent‑Tiering**        | Unknown/variable access               | Auto-tiers, *no retrieval fees*, small per-object monitoring fee |
| **Standard‑IA**                | Infrequent access but rapid retrieval | Lower \$/GB, 30‑day minimum, retrieval fees                      |
| **One Zone‑IA**                | Re-creatable data, single AZ          | Lowest IA price, 30‑day minimum, AZ risk                         |
| **Glacier Instant Retrieval**  | Archive with millisecond access       | 90‑day minimum, low \$/GB, retrieval fees                        |
| **Glacier Flexible Retrieval** | Archive hours-scale                   | 90‑day minimum, expedited/standard/bulk retrieval tiers          |
| **Glacier Deep Archive**       | Long-term cold archive                | 180‑day minimum, very low \$/GB                                  |
| **S3 Express One Zone**        | Ultra‑low latency, single AZ          | Separate pricing model; very fast PUT/GET                        |

---

## 📏 Cost formula (simple mental model)

```
Monthly S3 cost ≈ (Avg stored GB × $/GB‑month)
                + (Σ requests / 1,000 × price per 1,000)
                + (Data transfer out GB × $/GB)
                + (Any feature add‑ons: monitoring, replication, Object Lambda, etc.)
```

---

## 🧮 Worked examples (us-east-1)

### Example A — Small web app (hot data)

* 100 GB in **Standard** all month → `100 × $0.023 = $2.30`
* 500k **GET** → `500,000/1,000 × $0.0004 = $0.20`
* 20k **PUT/LIST** → `20,000/1,000 × $0.005 = $0.10`
* 50 GB data **egress** to internet (outside free allocations) → `50 × $/GB (per AWS egress price)`
* **Rough total** (excluding egress): **\$2.60** + egress

### Example B — Archive 10 TB for a year

* 10,240 GB in **Glacier Deep Archive** → `10,240 × $0.00099 ≈ $10.14 / month`
* Retrieval later will add **restore + retrieval + temporary Standard storage** during restore window.

> For precise egress/restore pricing, plug numbers into the **AWS Pricing Calculator**.

---

## ⚠️ Gotchas that often surprise teams

* **Minimums:** 30/90/180‑day minimum storage durations and **128 KB minimum billable object sizes** in some classes.
* **Request costs add up:** High small‑object workloads (many PUT/LIST/GET) can exceed storage cost.
* **Lifecycle transitions aren’t free:** You pay per 1,000 transitions.
* **Cross‑Region Replication:** You pay for **requests**, **replicated storage** in the destination, and **data transfer** between Regions.
* **Data transfer out to the internet** is billed separately from S3 storage (first check free/discounted allocations, then the standard \$/GB tiers).
* **Console browsing generates charges** (LIST/GET) just like using the API.

---

## ✅ How to keep S3 bills low

* **Right‑size storage class** (use lifecycle policies; send truly cold data to archive classes).
* **Bundle small files** (e.g., parquet/ORC, tar before archive) to reduce request overhead.
* **Enable Intelligent‑Tiering** when access is unpredictable.
* **Use CloudFront** for public downloads to reduce S3 GETs and egress patterns.
* **Prefer same‑Region** data flows; avoid unnecessary cross‑Region egress.
* **Measure** with **S3 Storage Lens**, **Cost & Usage Reports**, and **AWS Budgets/Alerts**.

---

## 🧰 Copy‑paste snippets

### Lifecycle: transition to IA after 30 days, Deep Archive after 180 days

```json
{
  "Rules": [
    {
      "ID": "TransitionToIAAndDeepArchive",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 180, "StorageClass": "DEEP_ARCHIVE"}
      ],
      "NoncurrentVersionTransitions": [
        {"NoncurrentDays": 30, "StorageClass": "STANDARD_IA"},
        {"NoncurrentDays": 180, "StorageClass": "DEEP_ARCHIVE"}
      ],
      "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
    }
  ]
}
```

### Cost tags (sample)

```json
{
  "TagSet": [
    {"Key": "Env", "Value": "prod"},
    {"Key": "Team", "Value": "platform"},
    {"Key": "CostCenter", "Value": "cc-1234"}
  ]
}
```

---

## 🔗 Official references

* S3 Pricing: [https://aws.amazon.com/s3/pricing/](https://aws.amazon.com/s3/pricing/)
* AWS Pricing Calculator (S3): [https://calculator.aws](https://calculator.aws)
* S3 Storage Classes: [https://aws.amazon.com/s3/storage-classes/](https://aws.amazon.com/s3/storage-classes/)
* S3 Developer Guide (requests): [https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html](https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html)
* S3 Storage Lens: [https://aws.amazon.com/s3/storage-lens/](https://aws.amazon.com/s3/storage-lens/)

---

## 📄 License

MIT

---

> *PRs welcome! Update numbers when AWS pricing changes or if your Region differs.*
