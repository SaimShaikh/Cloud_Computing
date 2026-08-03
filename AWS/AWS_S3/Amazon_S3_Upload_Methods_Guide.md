# Amazon S3 Upload Methods

## Overview

Amazon S3 supports multiple upload methods depending on the file size
and use case.

  -----------------------------------------------------------------------
  Upload Method                          Max Object Size Best For
  ------------------------- ---------------------------- ----------------
  Single PUT Upload                                 5 GB Small files

  Multipart Upload                                  5 TB Large files

  Presigned URL / POST            Depends on upload type Direct
                                                         browser/mobile
                                                         uploads
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 1. Single PUT Upload

## How it works

The client uploads the entire object in a single HTTP PUT request.

``` text
Client
   │
   │ PUT Object
   ▼
Amazon S3
```

## Best for

-   Documents
-   Images
-   Configuration files
-   Log files
-   Small backups
-   Files up to 5 GB

## Advantages

-   Simple implementation
-   Low overhead
-   Fast for small files

## Limitations

-   Maximum object size is 5 GB.
-   If the upload fails, the entire file must be uploaded again.
-   No parallel upload support.

### Example

Uploading:

-   report.pdf (20 MB)
-   image.png (5 MB)
-   log.txt (50 KB)

------------------------------------------------------------------------

# 2. Multipart Upload (Recommended for Large Files)

## How it works

Instead of uploading one large file, the client splits it into multiple
parts.

``` text
10 GB File
      │
      ▼
Split into Parts
 │   │   │
 ▼   ▼   ▼
P1  P2  P3 ... P20
 │   │   │
 └───┴───┘
      ▼
 Amazon S3
      │
      ▼
Complete Multipart Upload
```

Each part is uploaded independently.

## Example

A 10 GB file is divided into twenty 500 MB parts.

If Parts 1--15 upload successfully and Part 16 fails:

-   Parts 1--15 remain stored.
-   Only Parts 16--20 need to be uploaded after resuming.
-   Previously uploaded parts are not uploaded again.

## Best for

-   Videos
-   ISO images
-   Database backups
-   VM images
-   Files larger than 100 MB (recommended)
-   Files larger than 5 GB (required)

## Advantages

-   Resume interrupted uploads
-   Parallel uploads
-   Retry only failed parts
-   Better performance over unstable networks
-   Supports objects up to 5 TB

## Important Limits

  Property                                                    Value
  ------------------------- ---------------------------------------
  Maximum object size                                          5 TB
  Maximum number of parts                                    10,000
  Part size                   5 MB--5 GB (last part can be smaller)

------------------------------------------------------------------------

# 3. Presigned URL / Presigned POST Upload

## How it works

Your backend generates a temporary upload URL.

The browser or mobile application uploads directly to Amazon S3 without
sending the file through your application server.

``` text
Browser / Mobile App
        │
        ▼
   Presigned URL
        │
        ▼
    Amazon S3
```

## Best for

-   Profile picture uploads
-   Resume uploads
-   User documents
-   Customer portals
-   Mobile applications

## Advantages

-   Reduces backend server load
-   Saves bandwidth
-   Improves scalability
-   More secure because the URL expires automatically

------------------------------------------------------------------------

# Comparison

  Feature           Single PUT    Multipart Upload   Presigned URL / POST
  ----------------- ------------- ------------------ ----------------------------------------
  Max Size          5 GB          5 TB               Depends on underlying upload
  Resume Upload     No            Yes                Depends on implementation
  Parallel Upload   No            Yes                Supported when combined with Multipart
  Best For          Small files   Large files        Direct browser/mobile uploads

------------------------------------------------------------------------

# Which Upload Method Should You Choose?

  -----------------------------------------------------------------------
  File Size                 Recommended Method
  ------------------------- ---------------------------------------------
  1 KB -- 100 MB            Single PUT Upload

  100 MB -- 5 GB            Multipart Upload (recommended)

  More than 5 GB            Multipart Upload (required)

  Browser or Mobile Upload  Presigned URL / POST (use Multipart for large
                            files)
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# Real-World Examples

  -----------------------------------------------------------------------
  Scenario                Recommended Method
  ----------------------- -----------------------------------------------
  Upload a PDF            Single PUT

  Upload a profile        Presigned URL
  picture                 

  Upload a 20 GB database Multipart Upload
  backup                  

  Upload a 200 GB VM      Multipart Upload
  backup                  

  Upload a 15 GB video    Presigned URL + Multipart Upload
  from a website          
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# Interview Questions

### Q1. Can a Single PUT upload files larger than 5 GB?

**Answer:** No.

------------------------------------------------------------------------

### Q2. Which upload method supports resume?

**Answer:** Multipart Upload.

------------------------------------------------------------------------

### Q3. Who decides to use Multipart Upload?

**Answer:** The client (AWS CLI, AWS SDKs, AWS Management Console, or
your application). Amazon S3 provides the capability, but the client
decides when to use it.

------------------------------------------------------------------------

### Q4. What happens if a Multipart Upload fails at 75%?

**Answer:** Successfully uploaded parts remain stored. Only the failed
or missing parts need to be uploaded again.

------------------------------------------------------------------------

# Key Takeaways

-   Use **Single PUT** for small files.
-   Use **Multipart Upload** for large files or unreliable networks.
-   Use **Presigned URLs** for secure direct uploads from browsers and
    mobile applications.
-   Multipart Upload is required for objects larger than **5 GB** and
    supports objects up to **5 TB**.
