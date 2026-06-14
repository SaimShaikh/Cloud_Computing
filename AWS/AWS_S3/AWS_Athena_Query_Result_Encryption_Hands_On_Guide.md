# AWS Athena Hands-on Lab: Encrypt Query Results Stored in Amazon S3

> Production-style end-to-end lab (Part 1)

## Table of Contents

1.  Introduction
2.  Prerequisites
3.  Create IAM Admin User
4.  Login as IAM User
5.  Create S3 Buckets
6.  Upload Sample Data
7.  Open Athena
8.  Configure Query Result Location
9.  Create Database
10. Create Table
11. Run Queries
12. Enable SSE-S3
13. Verify Encryption
14. Enable SSE-KMS
15. Verify KMS Encryption
16. Troubleshooting
17. Cleanup

------------------------------------------------------------------------

# 1. Introduction

Athena executes SQL directly on data in S3. Every query produces an
output file that is stored in an S3 bucket. In this lab you will secure
those output files using server-side encryption.

------------------------------------------------------------------------

# 2. Prerequisites

-   AWS account
-   Billing enabled
-   Region: ap-south-1 (recommended)
-   Browser

------------------------------------------------------------------------

# 3. Create IAM Admin User

## Why?

Never perform daily work using the root account.

### Step 1

Login as Root.

### Step 2

Open IAM.

### Step 3

Create Group

Name:

    Admins

Attach policy:

    AdministratorAccess

### Step 4

Create User

    athena-admin

Enable console access.

Assign to:

    Admins

Download credentials.

------------------------------------------------------------------------

# 4. Login as IAM User

Logout.

Login using IAM URL and the created credentials.

------------------------------------------------------------------------

# 5. Create Source Bucket

Example:

    athena-source-yourname

Keep Block Public Access enabled.

Enable default encryption.

------------------------------------------------------------------------

# 6. Create Query Result Bucket

Example:

    athena-results-yourname

Enable default encryption.

------------------------------------------------------------------------

# 7. Upload Sample CSV

employees.csv

``` csv
id,name,department,salary
1,Alice,HR,50000
2,Bob,IT,75000
3,Charlie,Finance,65000
4,David,IT,82000
5,Eva,Marketing,58000
```

Upload to source bucket.

------------------------------------------------------------------------

# 8. Open Athena

First launch will ask for query result location.

Set:

    s3://athena-results-yourname/

Save.

------------------------------------------------------------------------

# 9. Create Database

``` sql
CREATE DATABASE company;
```

------------------------------------------------------------------------

# 10. Create External Table

``` sql
CREATE EXTERNAL TABLE employees(
id INT,
name STRING,
department STRING,
salary INT
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION 's3://athena-source-yourname/';
```

------------------------------------------------------------------------

# 11. Run Query

``` sql
SELECT * FROM employees;
```

Confirm CSV appears in results bucket.

------------------------------------------------------------------------

# 12. Enable SSE-S3

Athena → Settings → Encrypt query results → SSE-S3

Run:

``` sql
SELECT department,AVG(salary)
FROM employees
GROUP BY department;
```

Verify object properties show SSE-S3.

------------------------------------------------------------------------

# 13. Create KMS Key

Open KMS.

Create symmetric key.

Alias:

    athena-key

Allow yourself as key administrator and user.

Copy ARN.

------------------------------------------------------------------------

# 14. Enable SSE-KMS

Athena Settings

Enable encryption.

Choose SSE-KMS.

Paste key ARN.

Run:

``` sql
SELECT * FROM employees WHERE salary>60000;
```

Verify object uses AWS KMS.

------------------------------------------------------------------------

# 15. Internal Workflow

    User
      |
    SQL
      |
    Athena
      |
    Read S3
      |
    Generate Results
      |
    KMS Data Key
      |
    Encrypt
      |
    Store in S3

------------------------------------------------------------------------

# 16. Troubleshooting

## No output location

Configure Athena result bucket.

## AccessDenied

Check IAM and S3 permissions.

## KMS access denied

Update KMS key policy and IAM permissions.

------------------------------------------------------------------------

# 17. Cleanup

Delete:

-   Athena table
-   Database
-   Result bucket
-   Source bucket
-   Schedule KMS key deletion

------------------------------------------------------------------------

# Next Expansion

This guide is the foundation. A full production manual can further
include: - Glue Data Catalog deep dive - Workgroups - Bucket policies -
IAM least privilege - KMS key policies - CloudTrail auditing - Terraform
automation - AWS CLI implementation - Interview questions - Real
production architecture
