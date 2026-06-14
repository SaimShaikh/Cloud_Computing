# AWS Athena Query Result Encryption — Master Guide
### Ensure AWS Athena Query Results Stored in Amazon S3 Are Encrypted at Rest



---

## Chapter 1: Introduction

### What This Guide Covers

This guide teaches you how to ensure that AWS Athena query results stored in Amazon S3 are encrypted at rest. You will start from IAM setup, build S3 buckets, configure Athena, and enable both SSE-S3 and SSE-KMS encryption. By the end, you will understand not just the steps, but the reasoning, the architecture, and the internal mechanics.

### Why This Matters

When Amazon Athena executes a SQL query, it stores the result as a CSV file in an Amazon S3 bucket. If that bucket is unprotected, any user or service with S3 access can read those query results — including sensitive financial data, customer records, or proprietary analytics.

Encryption at rest ensures that even if someone gains access to the S3 storage layer, the data is unreadable without the correct decryption key.

### Lab Objective

By the end of this guide, you will be able to:

- Set up IAM users, groups, roles, and policies correctly
- Create and configure S3 buckets for Athena
- Upload and query sample data
- Enable SSE-S3 encryption on Athena query results
- Enable SSE-KMS encryption with a customer-managed key
- Enforce encryption through Athena Workgroups
- Verify encryption on result objects
- Audit all key usage through CloudTrail
- Automate the entire setup with Terraform and AWS CLI

### Overall Architecture

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/00c8e567-020e-4370-9a27-99e3ddf59151" />


---

## Chapter 2: AWS Global Infrastructure

### Regions and Availability Zones

AWS operates in Regions — geographic areas like `ap-south-1` (Mumbai), `us-east-1` (N. Virginia), `eu-west-1` (Ireland). Each Region contains multiple Availability Zones (AZs), which are physically separate data centers.

**For this lab, use one consistent Region throughout.** Recommended for India-based users: `ap-south-1` (Mumbai).

### Why Region Consistency Matters

- S3 buckets are regional
- KMS keys are regional — a key in `ap-south-1` cannot be used by Athena in `us-east-1`
- Athena is regional
- All resources in this lab must be in the same Region

### Global vs Regional Services

| Service | Scope |
|---------|-------|
| IAM | Global |
| S3 Bucket | Regional |
| KMS Keys | Regional |
| Athena | Regional |
| CloudTrail | Single-region or multi-region |

---

## Chapter 3: Root User Security

### Why Root User is Dangerous

The root user has unlimited access — it can delete your entire account, change billing, and bypass IAM. If its credentials are compromised, an attacker has full control.

### Secure Your Root User (Do This First)

**Step 1: Enable MFA on root user**
- Console → Top right → Account name → Security credentials
- Multi-factor authentication → Assign MFA device
- Use an authenticator app (Google Authenticator, Authy)

**Step 2: Do NOT create access keys for root**
- Root access keys are a critical security risk
- If any exist → delete them immediately

**Step 3: Never use root for daily work**
- Create an IAM admin user (Chapter 6)
- Use that IAM user for all tasks in this guide

**Step 4: Set billing alerts**
- Billing → Billing preferences → Enable billing alerts
- Notifies you of unexpected charges during lab work

---

## Chapter 4: IAM Fundamentals

### What is IAM?

AWS Identity and Access Management (IAM) controls who can do what in your AWS account. It is the permission system for every AWS service.

### Core Concepts

**User** — A person or application that interacts with AWS. Has long-term credentials (password or access key + secret key).

**Group** — A collection of users. Policies attached to a group apply to all users in it.

**Role** — An identity that AWS services (like Athena or Lambda) assume to gain permissions. Uses short-term credentials.

**Policy** — A JSON document defining what actions are allowed or denied on which resources.

### How Permissions Are Evaluated

```
Request comes in
      |
      v
Is there an explicit DENY anywhere? --> YES --> Access Denied (always wins)
      |
      NO
      v
Is there an explicit ALLOW? --> YES --> Access Granted
      |
      NO
      v
Default --> Access Denied (implicit deny)
```

### Example IAM Policy (S3 + Athena + KMS)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::your-athena-results-bucket",
        "arn:aws:s3:::your-athena-results-bucket/*",
        "arn:aws:s3:::your-source-data-bucket",
        "arn:aws:s3:::your-source-data-bucket/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "athena:StartQueryExecution",
        "athena:GetQueryExecution",
        "athena:GetQueryResults",
        "athena:ListWorkGroups",
        "athena:GetWorkGroup"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "glue:GetDatabase",
        "glue:GetTable",
        "glue:GetPartitions",
        "glue:CreateDatabase",
        "glue:CreateTable"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:ap-south-1:ACCOUNT_ID:key/KEY_ID"
    }
  ]
}
```

---

## Chapter 5: Create IAM Groups

### Why Groups?

Instead of attaching policies to individual users (which becomes unmanageable at scale), attach policies to groups and add users to those groups.

### Create an Admin Group

1. Open IAM Console
2. Left panel → **User groups** → **Create group**
3. Group name: `Administrators`
4. Attach policy: `AdministratorAccess` (AWS managed)
5. Click **Create group**

### Create a Restricted Analytics Group (Production Approach)

For production, create a group with limited permissions:

1. Create group: `AthenaAnalysts`
2. Attach policies:
   - `AmazonAthenaFullAccess`
   - Custom S3 policy scoped to specific buckets
   - Custom KMS policy for decrypt only (not key admin)

---

## Chapter 6: Create IAM Users

### Create Your Admin IAM User

1. IAM Console → **Users** → **Create user**
2. Username: `admin-yourname`
3. Select: **Provide user access to the AWS Management Console**
4. Set a password
5. Permissions → Add to group → Select `Administrators`
6. Click **Create user**
7. Download or save credentials

### Sign In as IAM User

- Go to: `https://ACCOUNT_ID.signin.aws.amazon.com/console`
- Find the sign-in URL in IAM → Dashboard

### Enable MFA on IAM User

Always enable MFA on admin users:
1. IAM → Users → Select your user
2. Security credentials tab → MFA device → Assign MFA device
3. Use an authenticator app

---

## Chapter 7: Create IAM Roles

### What is a Role?

A role is an identity that an AWS service assumes to get permissions. Athena needs permissions to read your S3 source data and write query results. When you run Athena from the console as an IAM user, Athena uses your IAM user's permissions. When called programmatically (from Lambda, EC2, ECS), the calling service must have a role.

### Create an Athena Execution Role (for programmatic use)

1. IAM → **Roles** → **Create role**
2. Trusted entity type: **AWS service**
3. Use case: Select **Athena** (or the service calling Athena)
4. Attach policies:
   - `AmazonAthenaFullAccess`
   - Custom S3 policy scoped to your buckets
   - Custom KMS policy for your key
5. Role name: `AthenaExecutionRole`
6. Click **Create role**

### Trust Policy (Who Can Assume This Role)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

---

## Chapter 8: IAM Policies Deep Dive

### Policy Types

| Type | Description |
|------|-------------|
| AWS Managed | Pre-built by AWS, maintained by AWS |
| Customer Managed | Created by you, fully under your control |
| Inline | Embedded directly in a user/role, not reusable |

### Least Privilege Principle

Always grant the minimum permissions required. For this lab:

**Required S3 permissions:**
- `s3:GetObject` on source bucket
- `s3:PutObject` on results bucket
- `s3:GetBucketLocation` on both
- `s3:ListBucket` on both

**Required KMS permissions:**
- `kms:GenerateDataKey` — Athena requests this to encrypt the result
- `kms:Decrypt` — Users need this to read encrypted results
- `kms:DescribeKey` — To verify key metadata

**Required Athena permissions:**
- `athena:StartQueryExecution`
- `athena:GetQueryExecution`
- `athena:GetQueryResults`

**Required Glue permissions:**
- `glue:GetDatabase`
- `glue:GetTable`
- `glue:GetPartitions`

### Condition Keys (Advanced)

Restrict KMS usage to only Athena:

```json
{
  "Effect": "Allow",
  "Action": "kms:GenerateDataKey",
  "Resource": "arn:aws:kms:ap-south-1:ACCOUNT_ID:key/KEY_ID",
  "Condition": {
    "StringEquals": {
      "kms:ViaService": "s3.ap-south-1.amazonaws.com"
    }
  }
}
```

This ensures the KMS key can only be used via S3 (not called directly), reducing the attack surface.

---

## Chapter 9: S3 Fundamentals

### What is Amazon S3?

Amazon Simple Storage Service (S3) is object storage — it stores files (objects) in buckets. Each object has a key (name/path), a value (file content), and metadata.

### Key Concepts

**Bucket** — A container for objects. Must be globally unique.

**Object** — A file stored in S3. Up to 5 TB per object.

**Key** — The full path of an object. Example: `results/2024/01/query-abc123.csv`

**Region** — Every bucket is created in a specific Region. Data stays there unless you configure replication.

### S3 Naming Rules

- 3–63 characters
- Lowercase letters, numbers, hyphens only
- No uppercase, no underscores
- Must start with a letter or number
- Must be globally unique across all AWS accounts

### Storage Classes

| Class | Use Case |
|-------|---------|
| S3 Standard | Frequently accessed data (Athena results) |
| S3 Standard-IA | Infrequently accessed |
| S3 Glacier | Archive — retrieval in minutes to hours |
| S3 Glacier Deep Archive | Long-term archive, cheapest |

---

## Chapter 10: S3 Bucket Policies

### What is a Bucket Policy?

A JSON document attached to an S3 bucket that controls access to that bucket and its objects. It applies to all identities (users, roles, services) that access the bucket.

### Enforce Encryption via Bucket Policy

This policy denies any PUT request that does not include SSE-KMS encryption:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedObjectUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::your-athena-results-bucket/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    },
    {
      "Sid": "DenyNonSecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::your-athena-results-bucket",
        "arn:aws:s3:::your-athena-results-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

Any upload without KMS encryption is rejected. Any HTTP (non-HTTPS) request is also rejected.

---

## Chapter 11: Server-Side Encryption Concepts

### What is Encryption at Rest?

Encryption at rest means data is encrypted when stored on disk. Even if someone physically accesses the storage hardware, they cannot read the data without the decryption key.

### Encryption Methods in AWS S3

| Method | Who Manages Keys | Audit | Cost |
|--------|-----------------|-------|------|
| SSE-S3 | AWS | None | Free |
| SSE-KMS | You (customer) | Full CloudTrail audit | KMS API charges |
| SSE-C | You (fully, you provide key on each request) | None | Free |
| CSE | You (encrypt before upload) | Optional | Depends |

### How SSE-S3 Works Internally

```
1. Athena sends PUT request with result CSV
2. S3 generates a unique data key for this object
3. S3 encrypts the object using AES-256
4. S3 stores encrypted object; key management is internal to S3
5. Object metadata marks it as SSE-S3 encrypted

On GET:
1. S3 retrieves its managed key
2. Decrypts and returns object to caller over HTTPS
```

### How SSE-KMS Works Internally

```
1. Athena sends PUT request (encryption: SSE-KMS + Key ARN)
2. S3 calls KMS: GenerateDataKey(KeyId=your-key-arn)
3. KMS returns: plaintext data key + encrypted copy of data key
4. S3 uses plaintext data key to encrypt object (AES-256 locally)
5. S3 discards the plaintext key immediately
6. S3 stores: encrypted object + encrypted data key in object metadata
7. CloudTrail logs the GenerateDataKey call

On GET:
1. User requests the object
2. S3 retrieves encrypted data key from object metadata
3. S3 calls KMS: Decrypt(encrypted-data-key)
4. KMS checks: does the caller have kms:Decrypt permission in IAM AND key policy?
5. If yes: KMS returns plaintext data key
6. S3 decrypts object and returns it to user
7. CloudTrail logs the Decrypt call
```

### SSE-S3 vs SSE-KMS Comparison

| Feature | SSE-S3 | SSE-KMS |
|---------|--------|---------|
| Key management | AWS | You |
| Audit trail | None | CloudTrail logs every use |
| Access control | S3 permissions only | S3 + KMS permissions both required |
| Key rotation | Automatic (AWS) | Configurable (manual or auto) |
| Compliance | Basic | PCI DSS, HIPAA, GDPR ready |
| Cross-account | Limited | Supported via key policy |
| Cost | Free | ~$0.03 per 10,000 KMS API requests |

---

## Chapter 12: AWS KMS Deep Dive

### What is AWS KMS?

AWS Key Management Service (KMS) is a managed service that creates and controls cryptographic keys used to encrypt data across AWS services.

### Key Types

| Type | Who Manages | Cost | Visibility |
|------|-------------|------|------------|
| AWS Managed Keys | AWS (auto-created by services) | Free | Visible in your account |
| Customer Managed Keys (CMK) | You | $1/month + API charges | Full control |
| AWS Owned Keys | AWS (internal use) | Free | Not visible |

### Envelope Encryption — How KMS Actually Works

KMS never directly encrypts your large files. It uses envelope encryption:

```
Your large file
        |
        v
KMS generates a Data Encryption Key (DEK)
        |
        |--- DEK encrypts your file locally (AES-256, fast)
        |
        |--- DEK is then encrypted by your KMS CMK
        |
Result stored: [Encrypted File] + [Encrypted DEK]

On Decrypt:
        |
KMS CMK decrypts the DEK
        |
DEK decrypts your file
        |
You receive the original file
```

Why this design?
- The CMK never leaves KMS hardware (FIPS 140-2 validated HSM)
- Your data is encrypted locally (fast, no size limit)
- KMS only handles the tiny DEK, not your entire file

### Key Rotation

For customer managed keys, enable automatic annual rotation:
- KMS creates new key material each year
- Old material is retained to decrypt previously encrypted data
- New data uses new material
- The key ID and ARN remain the same

Enable rotation:
```bash
aws kms enable-key-rotation --key-id YOUR_KEY_ID
```

---

## Chapter 13: KMS Key Policies

### What is a Key Policy?

Every KMS key has exactly one key policy. It is a resource-based policy (like an S3 bucket policy) that defines who can use and manage the key.

**Critical difference from IAM**: Both the IAM policy AND the key policy must allow an action for it to succeed. If either denies or omits the permission, access is denied.

### Default Key Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_ID:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    }
  ]
}
```

This allows the AWS account root to perform any KMS action, and delegates further access control to IAM policies.

### Key Policy for Athena Encryption Use Case

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_ID:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow key administrators",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_ID:user/admin-yourname"
      },
      "Action": [
        "kms:Create*", "kms:Describe*", "kms:Enable*",
        "kms:List*", "kms:Put*", "kms:Update*",
        "kms:Revoke*", "kms:Disable*", "kms:Delete*",
        "kms:ScheduleKeyDeletion", "kms:CancelKeyDeletion"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Allow Athena users to use the key",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_ID:user/admin-yourname"
      },
      "Action": [
        "kms:GenerateDataKey",
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Chapter 14: Athena Architecture

### What is Amazon Athena?

Amazon Athena is a serverless, interactive query service that lets you analyze data in S3 using standard SQL. No servers to provision, no clusters to manage, no idle cost.

### How Athena Works Internally

```
User submits SQL query
        |
        v
Athena Query Engine (Presto-based)
        |
        |-- Reads table metadata from AWS Glue Data Catalog
        |
        |-- Determines file locations in S3
        |
        |-- Reads source data files from S3 in parallel
        |
        |-- Executes SQL (filter, join, aggregate)
        |
        |-- Generates result set
        |
        |-- Writes result CSV to S3 results location
        |
        v
Returns query execution ID to caller
        |
Caller polls GetQueryResults to retrieve data
```

### Schema on Read

Athena uses schema on read — the data files in S3 have no embedded schema. The schema is defined in the Glue Data Catalog table definition, and applied when Athena reads the file. This differs from traditional databases where schema is enforced on write.

### Athena Pricing

- $5 per TB of data scanned
- Compressed, columnar formats (Parquet, ORC) reduce scanning and cost
- No charge for DDL queries (CREATE TABLE, DROP TABLE)
- No charge for failed queries

---

## Chapter 15: AWS Glue Data Catalog

### What is the Glue Data Catalog?

A central metadata repository. It stores database and table definitions — column names, data types, file formats, and S3 locations. Athena uses the Glue Data Catalog to understand your data structure without the files containing that information.

### Glue Concepts

**Database** — A logical grouping of tables (like a schema in traditional SQL).

**Table** — Metadata describing how to read a set of S3 files. Includes column definitions, file format (CSV, Parquet, JSON), and the S3 prefix.

**Partition** — A subdivision of a table mapped to a sub-path in S3. Example: `s3://bucket/data/year=2024/month=01/`

### How Athena Uses Glue

```
Query: SELECT * FROM employees WHERE salary > 60000

Step 1: Athena queries Glue Data Catalog
        - Database: lab_database
        - Table: employees
        - S3 location: s3://source-bucket/employees/
        - Format: CSV
        - Columns: id, name, department, salary, city

Step 2: Athena reads files from that S3 path
Step 3: Applies WHERE filter
Step 4: Returns matching rows + writes result CSV
```

---

## Chapter 16: Athena Workgroups

### What is a Workgroup?

A Workgroup is a logical container in Athena that groups users, queries, and settings. It lets you:
- Set a common query result location for all users
- Enforce encryption on all query results
- Set query cost controls (max bytes scanned per query)
- Separate different teams or environments

### Default Workgroup

Every AWS account has a `primary` workgroup. Unless you create others, all Athena queries run in `primary`.

### Why Workgroups Are Critical for Encryption

Without Workgroup enforcement, individual users can override encryption settings through personal Athena settings or via the API. By enabling **Override client-side settings**, you ensure ALL queries in that Workgroup use your defined encryption — users cannot opt out.

### Workgroup Settings for This Lab

| Setting | Purpose |
|---------|---------|
| Query result location | S3 path where all results are stored |
| Encrypt query results | Enables SSE-S3 or SSE-KMS |
| Override client-side settings | Prevents users from changing these settings |
| Bytes scanned per query limit | Cost control |

---

## Chapter 17: Create Source S3 Bucket

### Prerequisites

- Logged into AWS Console as your IAM admin user (not root)
- Region selected: `ap-south-1` (or your chosen region)

### Step 1: Open S3 Console

Search bar → type **S3** → click **S3**.

### Step 2: Create the Source Data Bucket

1. Click **Create bucket**
2. Bucket name: `athena-source-data-yourname-2024` *(globally unique)*
3. AWS Region: `ap-south-1`
4. Block Public Access: leave all 4 checkboxes checked
5. Bucket Versioning: Disable
6. Default encryption: **Amazon S3 managed keys (SSE-S3)** | Bucket Key: Enable
7. Click **Create bucket**

### Step 3: Create the Query Results Bucket

1. Click **Create bucket**
2. Bucket name: `athena-query-results-yourname-2024`
3. Same Region
4. Block all public access: Yes
5. Default encryption: **SSE-S3** (we upgrade to SSE-KMS in Chapter 26)
6. Click **Create bucket**

### Why Two Separate Buckets?

| Reason | Explanation |
|--------|-------------|
| Separation of concerns | Different access patterns and retention needs |
| Lifecycle policies | Delete results after 30 days; keep source data for years |
| Access control | Different teams may need access to each |
| Cost tracking | Easier to monitor per-bucket storage cost |

---

## Chapter 18: Upload Dataset

### Step 1: Create Sample CSV Data

Create a file called `employees.csv` on your local machine:

```
id,name,department,salary,city
1,Ravi Kumar,Engineering,75000,Pune
2,Priya Sharma,Marketing,52000,Mumbai
3,Arjun Mehta,Engineering,88000,Bengaluru
4,Sunita Rao,HR,45000,Hyderabad
5,Vikram Singh,Engineering,95000,Bengaluru
6,Ananya Patel,Finance,63000,Ahmedabad
7,Rohit Joshi,Marketing,58000,Pune
8,Neha Gupta,HR,47000,Mumbai
9,Karan Nair,Engineering,82000,Bengaluru
10,Divya Menon,Finance,71000,Hyderabad
```

### Step 2: Upload to S3

1. Open bucket: `athena-source-data-yourname-2024`
2. Click **Create folder** → name it `employees` → Create
3. Click into the `employees` folder
4. Click **Upload** → **Add files** → select `employees.csv`
5. Click **Upload**

### Verify

File should be visible at:
`s3://athena-source-data-yourname-2024/employees/employees.csv`

---

## Chapter 19: Configure Athena Settings

### Step 1: Open Athena Console

Search bar → **Athena** → click **Amazon Athena**.

### Step 2: Set Query Result Location

1. Click **Settings** → **Manage**
2. Query result location: `s3://athena-query-results-yourname-2024/`
3. Click **Save**

Without this, Athena throws: **No output location provided** and cannot complete any query.

### Step 3: Confirm Workgroup

Verify you are in the **primary** workgroup (shown in the top-right of the query editor). We will configure Workgroup encryption after the initial hands-on steps.

---

## Chapter 20: Create Athena Database

### Open Query Editor

Athena Console → **Query editor** (left panel).

### Create Database

```sql
CREATE DATABASE IF NOT EXISTS lab_database;
```

Click **Run**.

In the left panel under **Database**, select `lab_database` from the dropdown.

### What This Does

Creates a logical namespace in the AWS Glue Data Catalog. No files are created in S3. All tables you create next belong to this database.

---

## Chapter 21: Create Athena Tables

### Select the Database

In the Query Editor left panel, select `lab_database`.

### Create the Employees Table

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS lab_database.employees (
  id         INT,
  name       STRING,
  department STRING,
  salary     INT,
  city       STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
STORED AS TEXTFILE
LOCATION 's3://athena-source-data-yourname-2024/employees/'
TBLPROPERTIES ('skip.header.line.count'='1');
```

Click **Run**.

### What Each Clause Means

| Clause | Meaning |
|--------|---------|
| `CREATE EXTERNAL TABLE` | Metadata only — data stays in S3, not moved |
| `FIELDS TERMINATED BY ','` | CSV format |
| `skip.header.line.count=1` | Skip the first row (column names) |
| `LOCATION` | S3 folder containing the data files |

### EXTERNAL vs Managed Tables

**EXTERNAL**: Dropping the table removes metadata from Glue only — S3 files are NOT deleted.

**Managed (CTAS)**: Athena manages the data location. Dropping the table deletes the files. Use for derived tables, not source data.

---

## Chapter 22: Run SQL Queries

### Query 1: Select All

```sql
SELECT * FROM lab_database.employees;
```

Expected: 10 rows, all columns.

### Query 2: Filter by Salary

```sql
SELECT name, department, salary
FROM lab_database.employees
WHERE salary > 60000
ORDER BY salary DESC;
```

### Query 3: Department Average

```sql
SELECT department,
       AVG(salary) AS avg_salary,
       COUNT(*)    AS headcount
FROM lab_database.employees
GROUP BY department
ORDER BY avg_salary DESC;
```

### Query 4: Count Records

```sql
SELECT COUNT(*) AS total_employees
FROM lab_database.employees;
```

### Verify Result Files in S3

After each query:
1. Open `athena-query-results-yourname-2024`
2. You will see CSV files like `abc12345-6789-def0-ghij.csv`
3. Each query produces one CSV + one `.metadata` file

At this stage, result files use your bucket's default encryption (SSE-S3). We will now upgrade to explicitly configured encryption.

---

## Chapter 23: Enable SSE-S3 Encryption

### Method 1: Athena Settings (Per-User)

1. Athena Console → **Settings** → **Manage**
2. Encrypt query results: **Enable**
3. Encryption type: **SSE-S3**
4. Save

### Method 2: Workgroup (Enforced for All Users — Recommended)

1. Athena Console → **Workgroups** → select `primary` → **Edit**
2. Query result configuration:
   - Result location: `s3://athena-query-results-yourname-2024/`
   - Encrypt query results: **Enable**
   - Encryption: **SSE-S3**
3. **Override client-side settings**: **Enable**
4. Click **Save changes**

### Run Another Query

```sql
SELECT COUNT(*) FROM lab_database.employees;
```

The result file will now be encrypted with SSE-S3.

---

## Chapter 24: Verify SSE-S3 Encryption

### Step 1: Go to Results Bucket

Open `athena-query-results-yourname-2024`. Sort by Last Modified.

### Step 2: Open the Latest CSV File

Click the file name → click the **Properties** tab → scroll to **Server-side encryption**.

You should see:

```
Server-side encryption
Amazon S3 managed keys (SSE-S3)
```

### Step 3: Verify via CLI (Optional)

```bash
aws s3api head-object \
  --bucket athena-query-results-yourname-2024 \
  --key "your-result-file.csv"
```

Response:
```json
{
  "ServerSideEncryption": "AES256"
}
```

---

## Chapter 25: IAM + KMS Permissions

**Read this chapter before enabling SSE-KMS.** Understanding the permission model upfront prevents the most common errors.

### The Dual Permission Requirement

For SSE-KMS to work, the caller needs BOTH:

1. **S3 permission** to write/read the object (`s3:PutObject` / `s3:GetObject`)
2. **KMS permission** to use the key (`kms:GenerateDataKey` / `kms:Decrypt`)

If either is missing, the operation fails — even if the other permission is present.

```
User runs Athena query
        |
        v
Athena writes result to S3 [needs s3:PutObject]
        |
        v
S3 calls KMS to get encryption key [needs kms:GenerateDataKey]
        |
        v
Both must succeed. If either fails → AccessDenied


User reads result file
        |
        v
User calls S3 GetObject [needs s3:GetObject]
        |
        v
S3 calls KMS to decrypt the data key [needs kms:Decrypt]
        |
        v
Both must succeed. If either fails → AccessDenied
```

### IAM Policy for Athena + KMS User

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:GetBucketLocation",
        "s3:ListBucket",
        "s3:GetBucketAcl"
      ],
      "Resource": [
        "arn:aws:s3:::athena-source-data-yourname-2024",
        "arn:aws:s3:::athena-source-data-yourname-2024/*",
        "arn:aws:s3:::athena-query-results-yourname-2024",
        "arn:aws:s3:::athena-query-results-yourname-2024/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "athena:StartQueryExecution",
        "athena:GetQueryExecution",
        "athena:GetQueryResults",
        "athena:StopQueryExecution",
        "athena:ListWorkGroups",
        "athena:GetWorkGroup"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "glue:GetDatabase",
        "glue:GetTable",
        "glue:GetPartitions",
        "glue:CreateDatabase",
        "glue:CreateTable"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:GenerateDataKey",
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:ap-south-1:ACCOUNT_ID:key/KEY_ID"
    }
  ]
}
```

### Common Permission Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `KMS key not accessible` | Missing `kms:GenerateDataKey` in IAM policy | Add it to the user's policy |
| `AccessDenied on S3 PutObject` | Missing `s3:PutObject` | Add S3 permission |
| `AccessDenied when reading result` | Has S3 access but missing `kms:Decrypt` | Add `kms:Decrypt` to IAM policy |
| `KMS InvalidKeyUsage` | Used a signing key instead of encrypt/decrypt key | Create a Symmetric key |
| `KMS DisabledException` | Key was disabled | Re-enable the key in KMS console |

---

## Chapter 26: Enable SSE-KMS Encryption

### Step 1: Create a KMS Key

1. Search → **KMS** → **Key Management Service**
2. Left panel → **Customer managed keys** → **Create key**
3. Key type: **Symmetric**
4. Key usage: **Encrypt and decrypt**
5. Click **Next**
6. Alias: `athena-encryption-key`
7. Description: `Key for encrypting Athena query results`
8. Click **Next**
9. Key administrators: Select your IAM user
10. Key users: Select your IAM user
11. Click **Next** → **Finish**

### Step 2: Enable Key Rotation

1. Open the key you just created
2. Click the **Key rotation** tab
3. Toggle: **Automatically rotate this KMS key every year** → Enable

### Step 3: Copy the Key ARN

On the key details page, copy the **ARN**:
```
arn:aws:kms:ap-south-1:123456789012:key/abcd1234-5678-90ef-ghij-klmnopqrstuv
```

Save it — you need it next.

### Step 4: Configure Athena Workgroup for SSE-KMS

1. Athena → **Workgroups** → `primary` → **Edit**
2. Encrypt query results: **Enable**
3. Encryption type: **SSE-KMS**
4. KMS key: Select from dropdown or paste the ARN
5. Override client-side settings: **Enable**
6. Click **Save changes**

### Step 5: Run a Query

```sql
SELECT department, AVG(salary) AS avg_salary
FROM lab_database.employees
GROUP BY department;
```

---

## Chapter 27: Verify KMS Encryption

### Via S3 Console

1. Open `athena-query-results-yourname-2024`
2. Open the latest result CSV → **Properties** tab → **Server-side encryption**

You should see:
```
Server-side encryption
AWS Key Management Service key (SSE-KMS)
KMS key ARN: arn:aws:kms:ap-south-1:123456789012:key/abcd1234-...
```

### Via AWS CLI

```bash
aws s3api head-object \
  --bucket athena-query-results-yourname-2024 \
  --key "path/to/result.csv"
```

Response:
```json
{
  "ServerSideEncryption": "aws:kms",
  "SSEKMSKeyId": "arn:aws:kms:ap-south-1:123456789012:key/abcd1234-..."
}
```

### Via CloudTrail

1. Open **CloudTrail** → **Event history**
2. Filter: Event name = `GenerateDataKey`
3. You will see one event per Athena query result written

Each event shows: which IAM principal triggered it, which KMS key was used, and the exact timestamp.

### Test the Dual Permission Requirement

To prove that S3 access alone is not enough:

1. Create a test IAM user with ONLY `s3:GetObject` (no KMS permissions)
2. That user can see the file in S3 but gets `AccessDenied` when downloading
3. Add `kms:Decrypt` to that user's policy
4. The user can now download and read the file

This demonstrates that SSE-KMS adds a second, independent layer of access control.

---

## Chapter 28: Cross-Account Access

### Scenario

Data engineering team is in Account A. The analytics team runs Athena in Account B. You want Athena in Account B to query source data from Account A and write encrypted results using a KMS key owned by Account A.

### Configuration in Account A (Data Owner)

**S3 bucket policy on source bucket** — allow Account B's role to read:

```json
{
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::ACCOUNT_B_ID:role/AthenaRole"
  },
  "Action": ["s3:GetObject", "s3:ListBucket"],
  "Resource": [
    "arn:aws:s3:::source-bucket-account-a",
    "arn:aws:s3:::source-bucket-account-a/*"
  ]
}
```

**S3 bucket policy on results bucket** — allow Account B to write:

```json
{
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::ACCOUNT_B_ID:role/AthenaRole"
  },
  "Action": ["s3:PutObject", "s3:GetBucketLocation"],
  "Resource": [
    "arn:aws:s3:::results-bucket-account-a",
    "arn:aws:s3:::results-bucket-account-a/*"
  ]
}
```

**KMS key policy** — allow Account B's role to use the key:

```json
{
  "Sid": "Allow cross-account Athena access",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::ACCOUNT_B_ID:role/AthenaRole"
  },
  "Action": ["kms:GenerateDataKey", "kms:Decrypt", "kms:DescribeKey"],
  "Resource": "*"
}
```

### Configuration in Account B

IAM policy on `AthenaRole` — allow Athena actions and referencing cross-account resources:

```json
{
  "Effect": "Allow",
  "Action": [
    "athena:StartQueryExecution",
    "athena:GetQueryExecution",
    "athena:GetQueryResults"
  ],
  "Resource": "*"
},
{
  "Effect": "Allow",
  "Action": ["kms:GenerateDataKey", "kms:Decrypt", "kms:DescribeKey"],
  "Resource": "arn:aws:kms:ap-south-1:ACCOUNT_A_ID:key/KEY_ID"
}
```

All three layers (S3 bucket policy, KMS key policy, IAM policy) must be aligned for cross-account access to work.

---

## Chapter 29: CloudTrail Auditing

### What is CloudTrail?

AWS CloudTrail records every API call made in your account — who did what, when, and from where. With SSE-KMS, every Athena query result write and every authorized read generates KMS API calls that appear in CloudTrail. This is the audit trail.

### Enable CloudTrail

1. Search → **CloudTrail** → **Create trail**
2. Trail name: `management-trail`
3. Storage location: Create new S3 bucket for logs
4. Log file SSE-KMS encryption: Enable (best practice)
5. Management events: Read + Write
6. Click **Create trail**

### Key Events to Monitor

| Event | When It Fires | What It Shows |
|-------|--------------|---------------|
| `GenerateDataKey` | When Athena writes encrypted result | Who ran the query, which key used |
| `Decrypt` | When user reads encrypted result | Who accessed the data and when |
| `DisableKey` | When key is disabled | Potential security incident |
| `ScheduleKeyDeletion` | When key deletion is scheduled | Critical alert needed |

### Query CloudTrail Logs via Athena

After setting up CloudTrail to write logs to S3, you can query those logs with Athena:

```sql
SELECT eventtime, useridentity.arn, eventname, requestparameters
FROM cloudtrail_logs
WHERE eventsource = 'kms.amazonaws.com'
  AND eventname IN ('GenerateDataKey', 'Decrypt')
ORDER BY eventtime DESC
LIMIT 20;
```

---

## Chapter 30: Monitoring with CloudWatch

### Key Metrics to Monitor

| Metric | Namespace | Why |
|--------|-----------|-----|
| QueryExecutionCount | AWS/Athena | How many queries are running |
| DataScannedInBytes | AWS/Athena | Cost indicator |
| S3 PutRequests | AWS/S3 | Result files being written |
| S3 GetRequests | AWS/S3 | Result files being read |

### Create a CloudWatch Alarm for KMS Errors

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "AthenaKMSErrors" \
  --namespace "AWS/KMS" \
  --metric-name "Errors" \
  --statistic Sum \
  --period 300 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions "arn:aws:sns:ap-south-1:ACCOUNT_ID:your-alert-topic"
```

### CloudWatch Logs Insight Query for Unauthorized KMS Access

```
fields @timestamp, userIdentity.arn, eventName, errorCode
| filter eventSource = "kms.amazonaws.com"
| filter errorCode = "AccessDenied"
| sort @timestamp desc
| limit 50
```

---

## Chapter 31: Production Architecture

### Single-Account Production Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        AWS Account                           │
│                                                              │
│  ┌───────────┐   ┌──────────────┐   ┌────────────────────┐  │
│  │   Data    │   │  AWS Glue    │   │   Amazon Athena    │  │
│  │ Producer  │──▶│ Data Catalog │◀──│   Workgroup:       │  │
│  │  (ETL)    │   │  Databases   │   │   SSE-KMS enforced │  │
│  └───────────┘   │  Tables      │   └────────┬───────────┘  │
│       │          └──────────────┘            │              │
│       ▼                                      │              │
│  ┌──────────────────┐          ┌─────────────▼───────────┐  │
│  │  S3 Source Bucket│          │  S3 Results Bucket      │  │
│  │  (SSE-S3)        │◀─────────│  (SSE-KMS enforced)     │  │
│  └──────────────────┘  reads   └─────────────┬───────────┘  │
│                                              │              │
│                               ┌─────────────▼───────────┐  │
│                               │  AWS KMS (CMK)          │  │
│                               │  Rotation: Annual       │  │
│                               │  Audit: CloudTrail      │  │
│                               └─────────────────────────┘  │
│                                                              │
│  ┌──────────────┐   ┌──────────────┐                        │
│  │  CloudTrail  │   │  CloudWatch  │                        │
│  │  (KMS audit) │   │  (Alerts)    │                        │
│  └──────────────┘   └──────────────┘                        │
└──────────────────────────────────────────────────────────────┘
```

### Multi-Account Architecture (Enterprise)

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  Data Account   │    │ Security Account │    │Analytics Account│
│                 │    │                  │    │                 │
│  S3 Source Data │    │  AWS KMS CMK     │    │  Amazon Athena  │
│  (cross-account │◀───│  (key policy     │───▶│  Workgroup:     │
│  bucket policy) │    │   allows         │    │  cross-account  │
│                 │    │   analytics role)│    │  SSE-KMS key    │
│  S3 Results     │    │                  │    │                 │
│  (results       │    │  CloudTrail      │    │  IAM Role:      │
│   written here) │    │  (central audit) │    │  AthenaRole     │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

### Design Decisions

| Decision | Recommendation | Reason |
|----------|---------------|--------|
| SSE-S3 vs SSE-KMS | SSE-KMS always in production | Audit trail + dual access control |
| Key scope | One key per environment (dev/prod) | Isolate blast radius |
| Workgroup override | Always enable | Prevent accidental unencrypted results |
| S3 Bucket Key | Enable | Reduces KMS API costs by up to 99% |
| Key rotation | Enable annual rotation | Compliance requirement |
| CloudTrail | Multi-region trail | Catch activity in any region |

---

## Chapter 32: AWS CLI Commands

### Prerequisites

```bash
# Install AWS CLI
pip install awscli --upgrade --user

# Configure
aws configure
# AWS Access Key ID: [your key]
# AWS Secret Access Key: [your secret]
# Default region: ap-south-1
# Default output format: json
```

### S3 Commands

```bash
# Create source bucket
aws s3 mb s3://athena-source-data-yourname-2024 --region ap-south-1

# Create results bucket
aws s3 mb s3://athena-query-results-yourname-2024 --region ap-south-1

# Upload data
aws s3 cp employees.csv s3://athena-source-data-yourname-2024/employees/employees.csv

# List results bucket
aws s3 ls s3://athena-query-results-yourname-2024/ --recursive

# Check encryption on a specific object
aws s3api head-object \
  --bucket athena-query-results-yourname-2024 \
  --key "result-file.csv"
```

### KMS Commands

```bash
# Create a KMS key
aws kms create-key \
  --description "Athena query result encryption key" \
  --region ap-south-1

# Create alias
aws kms create-alias \
  --alias-name alias/athena-encryption-key \
  --target-key-id YOUR_KEY_ID \
  --region ap-south-1

# Get key ARN
aws kms describe-key \
  --key-id alias/athena-encryption-key \
  --query "KeyMetadata.Arn" \
  --output text

# Enable key rotation
aws kms enable-key-rotation --key-id YOUR_KEY_ID

# List all customer managed keys
aws kms list-keys --region ap-south-1
```

### Athena Commands

```bash
# Start a query with SSE-KMS encryption
aws athena start-query-execution \
  --query-string "SELECT COUNT(*) FROM lab_database.employees;" \
  --query-execution-context Database=lab_database \
  --result-configuration "OutputLocation=s3://athena-query-results-yourname-2024/,\
EncryptionConfiguration={EncryptionOption=SSE_KMS,\
KmsKey=arn:aws:kms:ap-south-1:ACCOUNT_ID:key/KEY_ID}" \
  --region ap-south-1

# Check query status
aws athena get-query-execution \
  --query-execution-id QUERY_EXECUTION_ID \
  --region ap-south-1

# Get query results
aws athena get-query-results \
  --query-execution-id QUERY_EXECUTION_ID \
  --region ap-south-1

# Update workgroup to enforce encryption
aws athena update-work-group \
  --work-group primary \
  --configuration-updates "ResultConfigurationUpdates={\
OutputLocation=s3://athena-query-results-yourname-2024/,\
EncryptionConfiguration={EncryptionOption=SSE_KMS,\
KmsKey=arn:aws:kms:ap-south-1:ACCOUNT_ID:key/KEY_ID}},\
EnforceWorkGroupConfiguration=true" \
  --region ap-south-1
```

---

## Chapter 33: Terraform Automation

### Project Structure

```
athena-encryption/
├── main.tf
├── variables.tf
├── outputs.tf
└── terraform.tfvars
```

### variables.tf

```hcl
variable "region" {
  default = "ap-south-1"
}

variable "account_id" {
  description = "Your AWS Account ID"
}

variable "unique_suffix" {
  description = "Unique suffix for S3 bucket names"
  default     = "yourname-2024"
}
```

### main.tf

```hcl
provider "aws" {
  region = var.region
}

# S3 — Source Data Bucket
resource "aws_s3_bucket" "source" {
  bucket = "athena-source-data-${var.unique_suffix}"
}

resource "aws_s3_bucket_public_access_block" "source" {
  bucket                  = aws_s3_bucket.source.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "source" {
  bucket = aws_s3_bucket.source.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# KMS Key
resource "aws_kms_key" "athena" {
  description             = "KMS key for Athena query result encryption"
  deletion_window_in_days = 7
  enable_key_rotation     = true

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      }
    ]
  })
}

resource "aws_kms_alias" "athena" {
  name          = "alias/athena-encryption-key"
  target_key_id = aws_kms_key.athena.key_id
}

# S3 — Results Bucket with SSE-KMS
resource "aws_s3_bucket" "results" {
  bucket = "athena-query-results-${var.unique_suffix}"
}

resource "aws_s3_bucket_public_access_block" "results" {
  bucket                  = aws_s3_bucket.results.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "results" {
  bucket = aws_s3_bucket.results.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.athena.arn
    }
    bucket_key_enabled = true
  }
}

# S3 Bucket Policy — Deny unencrypted uploads
resource "aws_s3_bucket_policy" "results_enforce_encryption" {
  bucket = aws_s3_bucket.results.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyUnencryptedUploads"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:PutObject"
        Resource  = "${aws_s3_bucket.results.arn}/*"
        Condition = {
          StringNotEquals = {
            "s3:x-amz-server-side-encryption" = "aws:kms"
          }
        }
      }
    ]
  })
}

# Athena Workgroup
resource "aws_athena_workgroup" "encrypted" {
  name = "encrypted-workgroup"

  configuration {
    enforce_workgroup_configuration    = true
    publish_cloudwatch_metrics_enabled = true

    result_configuration {
      output_location = "s3://${aws_s3_bucket.results.bucket}/"

      encryption_configuration {
        encryption_option = "SSE_KMS"
        kms_key_arn       = aws_kms_key.athena.arn
      }
    }
  }
}

# Athena Database
resource "aws_athena_database" "lab" {
  name   = "lab_database"
  bucket = aws_s3_bucket.results.bucket
}
```

### outputs.tf

```hcl
output "source_bucket"   { value = aws_s3_bucket.source.bucket }
output "results_bucket"  { value = aws_s3_bucket.results.bucket }
output "kms_key_arn"     { value = aws_kms_key.athena.arn }
output "kms_key_alias"   { value = aws_kms_alias.athena.name }
output "workgroup_name"  { value = aws_athena_workgroup.encrypted.name }
```

### Deploy

```bash
terraform init
terraform plan -var="account_id=YOUR_ACCOUNT_ID"
terraform apply -var="account_id=YOUR_ACCOUNT_ID"
terraform destroy  # cleanup when done
```

---

## Chapter 34: CI/CD Integration

### GitHub Actions Pipeline

```yaml
name: Deploy Athena Encryption Config

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2

      - name: Terraform Init
        run: terraform init
        working-directory: ./athena-encryption

      - name: Terraform Plan
        run: terraform plan -var="account_id=${{ secrets.AWS_ACCOUNT_ID }}"
        working-directory: ./athena-encryption

      - name: Terraform Apply
        run: terraform apply -auto-approve -var="account_id=${{ secrets.AWS_ACCOUNT_ID }}"
        working-directory: ./athena-encryption

      - name: Verify Workgroup Encryption Config
        run: |
          aws athena get-work-group --work-group encrypted-workgroup \
            --query "WorkGroup.Configuration.ResultConfiguration.EncryptionConfiguration"
```

---

## Chapter 35: Troubleshooting

### Error: "No output location provided"

**Cause**: Athena does not know where to write results.

**Fix**:
```bash
aws athena update-work-group \
  --work-group primary \
  --configuration-updates "ResultConfigurationUpdates={OutputLocation=s3://your-results-bucket/}"
```
Or set it in Athena Console → Settings → Manage.

---

### Error: "KMS key not accessible"

**Cause**: IAM user lacks `kms:GenerateDataKey`, or the KMS key policy does not include the principal.

**Fix**:
1. Check IAM policy — add `kms:GenerateDataKey` and `kms:Decrypt`
2. Check KMS Key Policy — add the user/role as a key user
3. Verify the key is in the same Region as Athena

---

### Error: "AccessDenied" when reading result file

**Cause**: User has S3 access but not `kms:Decrypt`.

**Fix**: Add `kms:Decrypt` and `kms:DescribeKey` to the user's IAM policy.

---

### Error: "Unable to verify/create output bucket"

**Cause**: Bucket does not exist in the correct Region, or user lacks `s3:GetBucketLocation`.

**Fix**:
1. Verify bucket name and Region match
2. Add `s3:GetBucketLocation` to the IAM policy

---

### Error: "HIVE_METASTORE_ERROR: Database does not exist"

**Cause**: Database not created or created in a different Region.

**Fix**: Run `CREATE DATABASE lab_database;` in the Athena Query Editor.

---

### Table Returns 0 Rows

**Cause**: The `LOCATION` in CREATE TABLE does not match the actual S3 path.

**Fix**: The LOCATION must point to the folder containing the CSV, not the CSV file itself. Drop and recreate the table with the correct path.

---

### Old Results Not Encrypted After Enabling SSE-KMS

**Cause**: Enabling SSE-KMS only applies to new results. Existing objects are not retroactively encrypted.

**Fix**: Re-run your queries. New result files will be encrypted. To encrypt existing objects, use S3 Batch Operations or copy them back to the same bucket with the new encryption header.

---

## Chapter 36: Cost Optimization

### Cost Drivers

| Item | Cost |
|------|------|
| Athena — data scanned | $5 per TB |
| S3 storage for results | ~$0.023/GB/month (Standard) |
| KMS API requests | ~$0.03 per 10,000 requests |
| KMS key | $1/month per CMK |

### Strategy 1: Enable S3 Bucket Key

Reduces KMS API calls by up to 99% by generating a per-bucket derived key from your CMK instead of calling KMS for every object.

Enable in S3 → Bucket → Properties → Default encryption → **Bucket Key: On**

Or in Terraform (already included in Chapter 33):
```hcl
bucket_key_enabled = true
```

### Strategy 2: Use Columnar Formats

Convert CSV source data to Parquet or ORC. Athena only reads the columns needed for a query.

```sql
CREATE TABLE employees_parquet
WITH (
  format = 'PARQUET',
  external_location = 's3://athena-source-data-yourname-2024/employees-parquet/'
)
AS SELECT * FROM employees;
```

A typical analytics query on Parquet scans 10–20x less data than CSV → 10–20x lower Athena cost.

### Strategy 3: Partition Your Data

```sql
CREATE EXTERNAL TABLE transactions_partitioned (
  id INT, amount DECIMAL, customer STRING
)
PARTITIONED BY (year STRING, month STRING)
STORED AS PARQUET
LOCATION 's3://your-bucket/transactions/';
```

Queries with `WHERE year='2024' AND month='01'` scan only that partition.

### Strategy 4: Set Result Expiry via S3 Lifecycle

Old query results accumulate and cost storage. Set a Lifecycle rule to delete them:

S3 → Bucket → Management → Lifecycle rules → Create rule → Expire objects after 30 days

---

## Chapter 37: Security Best Practices

### Key Principles

**1. Always use SSE-KMS in production**
SSE-S3 offers no audit trail and no fine-grained access control. SSE-KMS provides both.

**2. Enable key rotation**
```bash
aws kms enable-key-rotation --key-id YOUR_KEY_ID
```

**3. Enforce encryption via Workgroup**
Individual settings can be overridden. Workgroup with `EnforceWorkGroupConfiguration=true` cannot.

**4. Enable S3 Bucket Key**
Reduces KMS API costs by up to 99% without weakening security.

**5. Block all public access**
Both source and results buckets should have Block Public Access enabled.

**6. Deny HTTP access to S3**
Use bucket policy to deny `aws:SecureTransport: false`.

**7. Monitor with CloudTrail + CloudWatch**
Set alarms for unusual `kms:Decrypt` activity.

**8. Apply least privilege IAM**
Scope S3 permissions to specific bucket ARNs. Scope KMS permissions to specific key ARNs.

**9. Use resource-based policies on KMS keys**
Key policies are a second, independent layer of access control.

**10. Audit IAM credentials regularly**
Rotate access keys every 90 days. Remove unused users and roles.

### Production Security Checklist

- [ ] SSE-KMS enabled on Athena Workgroup
- [ ] Override client-side settings = ON
- [ ] Both S3 buckets block all public access
- [ ] S3 bucket policy denies unencrypted uploads
- [ ] S3 bucket policy denies HTTP access
- [ ] KMS key rotation enabled
- [ ] S3 Bucket Key enabled
- [ ] CloudTrail enabled for KMS events
- [ ] IAM users have least-privilege policies
- [ ] No root user access keys exist
- [ ] MFA on all admin users

---

## Chapter 38: Interview Questions

### Beginner Level

**Q1: What is Amazon Athena?**
Athena is a serverless, interactive query service from AWS that lets you analyze data stored in S3 using standard SQL. No infrastructure to manage — you write SQL and pay only for the data scanned.

---

**Q2: What is encryption at rest?**
Data is encrypted when stored on disk. Even if someone physically accesses the storage hardware, the data is unreadable without the decryption key. Different from encryption in transit, which protects data moving over the network.

---

**Q3: What is SSE-S3?**
Server-Side Encryption with S3-Managed Keys. S3 automatically encrypts objects using AES-256. AWS manages the keys entirely. Free, no extra configuration. No audit trail for individual object access.

---

**Q4: What is SSE-KMS?**
Server-Side Encryption with AWS KMS keys. You use a Customer Managed Key. Benefits: audit trail in CloudTrail, dual access control (S3 + KMS permissions both required), key rotation control, cross-account support. Small additional cost for KMS API calls.

---

**Q5: Where does Athena store query results?**
Athena stores each query result as a CSV file in a designated S3 bucket. The location is configured in Athena settings or the Workgroup configuration. A `.metadata` file is also created alongside each result.

---

### Intermediate Level

**Q6: How does envelope encryption work?**
KMS uses envelope encryption. Instead of encrypting your large data file directly with the CMK (slow, requires sending all data to KMS), KMS generates a small Data Encryption Key (DEK). S3 uses the DEK to encrypt your data locally using AES-256. The DEK is then encrypted by the CMK and stored alongside the object as metadata. On decrypt, KMS decrypts the DEK, and the DEK decrypts the object. The CMK never directly touches your data and never leaves KMS hardware.

---

**Q7: What is the difference between SSE-S3 and SSE-KMS?**

| Feature | SSE-S3 | SSE-KMS |
|---------|--------|---------|
| Key management | AWS | Customer |
| Audit | None | CloudTrail |
| Access control | S3 only | S3 + KMS both required |
| Cross-account | Limited | Yes |
| Compliance | Basic | HIPAA, PCI DSS, GDPR |
| Cost | Free | $1/month/key + API charges |

---

**Q8: What is an Athena Workgroup and why use it for encryption?**
A Workgroup groups Athena users and query settings. For encryption, enabling `EnforceWorkGroupConfiguration=true` makes the Workgroup's encryption settings override any individual user settings. Users cannot accidentally or intentionally run unencrypted queries. It is the production-safe way to guarantee all query results are always encrypted.

---

**Q9: A user has S3 GetObject permission on the results bucket. Can they read an SSE-KMS encrypted result?**
Not necessarily. With SSE-KMS, the user needs both S3 GetObject AND `kms:Decrypt` on the key that encrypted the object. If either permission is missing, access is denied. This is a key advantage of SSE-KMS over SSE-S3.

---

**Q10: How would you verify that an Athena query result is encrypted?**
Three ways:
1. S3 Console → open result CSV → Properties tab → Server-side encryption field
2. AWS CLI: `aws s3api head-object --bucket BUCKET --key FILE` — response includes `ServerSideEncryption` and `SSEKMSKeyId`
3. CloudTrail: search for `GenerateDataKey` events from Athena confirming KMS was invoked when the result was written

---

### Advanced Level

**Q11: What happens to old unencrypted Athena results after you enable SSE-KMS?**
Nothing automatically. SSE-KMS only applies to future results. Existing objects remain unencrypted unless you explicitly re-encrypt them using S3 Batch Operations or by copying each object back to the same location with the new encryption header.

---

**Q12: How do you enforce encryption cross-account?**
Three layers must all be aligned: the KMS key policy in Account A must allow Account B's role to use the key; the S3 bucket policy in Account A must allow Account B's role to read/write; and Account B's IAM role must have a policy permitting those actions. If any layer is missing, access fails.

---

**Q13: What is the cost impact of SSE-KMS for a team running 10,000 Athena queries per day?**
Each write makes one `GenerateDataKey` call. At 10,000/day = ~300,000/month. KMS charges ~$0.03 per 10,000 = ~$0.90/month. With S3 Bucket Key enabled, KMS calls are dramatically reduced (one key per bucket prefix, not per object), bringing costs near zero. Plus $1/month for the key itself.

---

**Q14: What is an S3 Bucket Key?**
A temporary, S3-side key derived from your CMK. Instead of calling KMS for every object, S3 uses the Bucket Key to encrypt multiple objects locally. The Bucket Key is refreshed periodically via a single KMS API call. Can reduce KMS API request costs by up to 99% with no weakening of encryption.

---

**Q15: Can you use a single KMS key for multiple Athena Workgroups?**
Yes. A KMS key can be used by multiple services, workgroups, and accounts — as long as the key policy and IAM policies allow it. In highly regulated environments, you might use separate keys per workgroup or data classification level to maintain stricter isolation in CloudTrail audit logs and limit blast radius if a key is compromised.

---

## Chapter 39: Mini Project

### Project: Encrypted Employee Analytics

**Goal**: End-to-end pipeline where all Athena query results are SSE-KMS encrypted and audited via CloudTrail.

**Steps**:
1. Set up infrastructure (S3 + KMS + Workgroup) using CLI commands from Chapter 32
2. Upload `employees.csv` from Chapter 18
3. Create database and table from Chapters 20–21
4. Run 5 analytical queries (by department, salary range, city, headcount, top earners)
5. Verify ALL 5 result files show SSE-KMS encryption in S3 Properties
6. Open CloudTrail → find 5 `GenerateDataKey` events
7. Create a second IAM user with no KMS permission → try to read a result → confirm `AccessDenied` → add `kms:Decrypt` → confirm access

**Success Criteria**:
- All 5 results show `aws:kms` in S3 Properties
- CloudTrail shows 5 KMS GenerateDataKey events
- Access control demonstration works as described

---

## Chapter 40: Enterprise Project

### Project: Secure Multi-Department Analytics Platform

**Scenario**: Three departments — Finance (PCI DSS), HR (GDPR), Engineering (internal). Each runs Athena queries on their own data with appropriate encryption controls.

**Requirements**:
- Finance: Dedicated Workgroup, dedicated KMS key, SSE-KMS mandatory, full CloudTrail
- HR: Dedicated Workgroup, dedicated KMS key, SSE-KMS mandatory
- Engineering: Shared Workgroup, SSE-S3 acceptable

**Deliverables**:

1. Terraform module creating:
   - 3 S3 source buckets + 3 results buckets
   - 2 KMS keys (Finance, HR)
   - 3 Athena Workgroups with appropriate encryption
   - 3 IAM roles scoped to each department
   - Bucket policies enforcing encryption for Finance and HR

2. IAM Role structure:
   - `FinanceAthenaRole`: access only Finance buckets + Finance KMS key
   - `HRAthenaRole`: access only HR buckets + HR KMS key
   - `EngineeringAthenaRole`: access only Engineering buckets, no KMS needed

3. CloudTrail: Single multi-region trail, KMS events forwarded to CloudWatch log group, alert for Decrypt calls outside business hours (18:00–08:00 IST)

4. Validation script: Shell script that runs a test query per workgroup and verifies result file encryption matches expected setting

---

## Chapter 41: Mastery Checklist

Use this to confirm you fully understand and can apply all concepts independently.

### Conceptual Understanding

- [ ] I can explain what Amazon Athena is and how it works (schema on read, serverless, Presto engine)
- [ ] I can explain the difference between SSE-S3 and SSE-KMS with their trade-offs
- [ ] I can explain envelope encryption and why KMS does not directly encrypt large files
- [ ] I can explain why S3 access alone is insufficient to read an SSE-KMS encrypted result
- [ ] I can explain what a KMS key policy is and how it differs from an IAM policy
- [ ] I can explain what an Athena Workgroup is and why it is used for encryption enforcement
- [ ] I can explain the CloudTrail audit trail for KMS events and what it shows

### Hands-On Skills

- [ ] I can create S3 buckets with proper public access blocking
- [ ] I can upload data to S3 and create an Athena external table pointing to that data
- [ ] I can create a Customer Managed Key in KMS with alias, key policy, and rotation enabled
- [ ] I can enable SSE-S3 on Athena query results and verify it
- [ ] I can enable SSE-KMS on Athena query results using a CMK an
