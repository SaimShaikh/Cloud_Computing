
# Amazon S3 Upload Methods

> This guide explains **how Amazon S3 uploads objects**, which upload
> method to choose, internal workflows, limits, performance
> considerations, and interview questions.

------------------------------------------------------------------------

# Table of Contents

1.  Introduction
2.  Single PUT Upload
3.  Multipart Upload
4.  Presigned URL & Presigned POST
5.  Multipart Upload API Workflow
6.  How Failed Uploads Are Handled
7.  Performance Best Practices
8.  S3 Upload Limits
9.  AWS CLI & SDK Behavior
10. Comparison Table
11. Real-World Scenarios
12. Interview Questions
13. Best Practices

------------------------------------------------------------------------

# Introduction

Amazon S3 stores every file as an **object** inside a **bucket**.

An object consists of:

-   Object Key (file name/path)
-   Object Data
-   Metadata
-   Version ID (if versioning is enabled)
-   ETag (checksum identifier)

S3 can store objects from **0 bytes to 5 TB**.

Depending on file size and application architecture, different upload
methods are available.

------------------------------------------------------------------------

# 1. Single PUT Upload

## Overview

A Single PUT Upload sends the complete object in one HTTP PUT request.

``` text
Application
      |
      | PUT Object
      v
 Amazon S3
```

### Maximum Size

**5 GB**

### Suitable For

-   Images
-   PDFs
-   Documents
-   Configuration files
-   Log files

### Advantages

-   Easy implementation
-   Less API overhead
-   Ideal for small files

### Limitations

-   No resume capability
-   No parallel upload
-   Entire upload restarts if interrupted

------------------------------------------------------------------------

# 2. Multipart Upload

## What is Multipart Upload?

Instead of uploading one large object, the client divides the object
into multiple independent parts.

Example:

``` text
10 GB File

Part 1 500 MB
Part 2 500 MB
...
Part 20 500 MB
```

Every part is uploaded separately.

After every part is uploaded, Amazon S3 combines them into one object.

------------------------------------------------------------------------

## Internal Workflow

``` text
Client
   |
   | InitiateMultipartUpload
   |
Upload ID Created
   |
Upload Part 1
Upload Part 2
Upload Part 3
...
Upload Part N
   |
CompleteMultipartUpload
   |
Amazon S3 Creates Final Object
```

------------------------------------------------------------------------

## Multipart Upload APIs

### Step 1 -- Initiate Multipart Upload

S3 returns an Upload ID.

Example

    Upload ID
    abc123xyz

------------------------------------------------------------------------

### Step 2 -- Upload Parts

Each request contains

-   Upload ID
-   Part Number
-   Part Data

Example

    UploadId=abc123xyz

    PartNumber=1
    PartNumber=2
    PartNumber=3

Each uploaded part returns an **ETag**.

The ETag is required during completion.

------------------------------------------------------------------------

### Step 3 -- Complete Multipart Upload

The client sends

-   Upload ID
-   Part Numbers
-   ETags

S3 validates every part and assembles the final object.

------------------------------------------------------------------------

## Failed Upload Example

Suppose

10 GB File

20 Parts

After uploading

    Part1 ✓
    Part2 ✓
    ...
    Part15 ✓

    Part16 Failed

Network disconnects.

When upload resumes

Only

    Part16
    Part17
    Part18
    Part19
    Part20

are uploaded.

Previously uploaded parts remain stored.

------------------------------------------------------------------------

## If Upload Never Completes

Incomplete parts remain inside S3.

They are billed as storage until:

-   Multipart upload completes
-   Multipart upload is aborted
-   Lifecycle Rule deletes incomplete uploads

Example Lifecycle Rule

Abort incomplete multipart uploads after **7 days**.

------------------------------------------------------------------------

## Parallel Upload

Multipart Upload allows multiple parts simultaneously.

``` text
Part1 --->

Part2 --->

Part3 --->

Part4 --->

Amazon S3
```

Benefits

-   Better bandwidth utilization
-   Faster uploads
-   Reduced upload time

------------------------------------------------------------------------

# 3. Presigned URL & Presigned POST

## Why use it?

Without presigned URLs

``` text
Browser
   |
Backend
   |
Amazon S3
```

Every upload passes through your server.

With Presigned URL

``` text
Browser

    |

Amazon S3
```

The backend only generates a temporary URL.

The browser uploads directly.

Benefits

-   Less backend load
-   Lower bandwidth costs
-   Better scalability

------------------------------------------------------------------------

# Upload Limits

  Property                                   Value
  --------------------- --------------------------
  Maximum object size                         5 TB
  Single PUT                                  5 GB
  Maximum parts                             10,000
  Minimum part size                           5 MB
  Maximum part size                           5 GB
  Last part               Can be smaller than 5 MB

------------------------------------------------------------------------

# AWS CLI Behavior

Example

    aws s3 cp backup.iso s3://my-bucket/

For small files

AWS CLI performs a normal PUT.

For large files

AWS CLI automatically switches to Multipart Upload based on its
multipart threshold (configurable).

AWS SDKs provide similar functionality.

------------------------------------------------------------------------

# Comparison

  Feature    Single PUT    Multipart     Presigned URL
  ---------- ------------- ------------- ----------------------
  Resume     No            Yes           Depends
  Parallel   No            Yes           Yes (with multipart)
  Max Size   5 GB          5 TB          Depends
  Best Use   Small files   Large files   Browser/Mobile

------------------------------------------------------------------------

# Real-World Scenarios

  Scenario                   Method
  -------------------------- ---------------------------
  Profile picture            Presigned URL
  PDF                        Single PUT
  50 GB Database Backup      Multipart
  VMware Image               Multipart
  CCTV Footage               Multipart
  User uploads 10 GB video   Presigned URL + Multipart

------------------------------------------------------------------------

# Best Practices

-   Use Single PUT only for small objects.
-   Use Multipart Upload for anything larger than 100 MB.
-   Multipart Upload is mandatory for objects larger than 5 GB.
-   Abort incomplete multipart uploads using Lifecycle Rules.
-   Validate uploads using ETags or checksums.
-   Use Presigned URLs for browser and mobile uploads.
-   Enable Transfer Acceleration for global users when beneficial.
-   Consider S3 Express One Zone for ultra-low-latency workloads where
    appropriate.

------------------------------------------------------------------------

# Interview Questions

**Q. Why is Multipart Upload faster?**

Because multiple parts can upload in parallel and failed parts are
retried independently.

**Q. Does S3 automatically split files?**

No. The client (AWS CLI, SDK, Console, or your application) decides to
use Multipart Upload.

**Q. What is Upload ID?**

A unique identifier returned when initiating a multipart upload. It
associates every uploaded part with the same upload session.

**Q. What is an ETag?**

A value returned after uploading each part. It is used to verify
uploaded parts and is required to complete a multipart upload.

**Q. What happens if CompleteMultipartUpload is never called?**

Uploaded parts remain incomplete and continue to incur storage charges
until they are aborted or cleaned up by a Lifecycle Rule.

------------------------------------------------------------------------

# Summary

  -----------------------------------------------------------------------
  File Size                 Recommended Upload
  ------------------------- ---------------------------------------------
  \<100 MB                  Single PUT

  100 MB--5 GB              Multipart Upload (recommended)

  \>5 GB                    Multipart Upload (required)

  Browser/Mobile            Presigned URL or Presigned POST (Multipart
                            for large files)
  -----------------------------------------------------------------------
