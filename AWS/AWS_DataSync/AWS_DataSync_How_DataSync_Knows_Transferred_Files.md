# How AWS DataSync Knows Which Data Has Already Been Transferred

## Overview

A common question about AWS DataSync is:

> **How does DataSync know which files have already been copied and
> which files still need to be transferred?**

The answer is:

**AWS DataSync does not maintain a permanent database of transferred
files.** Instead, every time a task runs, DataSync scans both the source
and destination, compares them, and transfers only new or changed data.

------------------------------------------------------------------------

# How DataSync Works

``` text
        Source Storage
              │
              ▼
      Scan File Metadata
              │
              ▼
      Scan Destination
              │
              ▼
      Compare Metadata
              │
      ┌───────┴────────┐
      │                │
   Same File      Changed/New File
      │                │
     Skip          Transfer
```

------------------------------------------------------------------------

# Step 1 -- Scan the Source

Example source directory:

``` text
/project

file1.pdf
file2.docx
file3.jpg
```

The DataSync Agent reads metadata such as:

-   File name/path
-   File size
-   Last modified timestamp
-   File permissions
-   Ownership (where supported)

Example:

  File         Size    Modified Time   Permissions
  ------------ ------- --------------- -------------
  file1.pdf    20 MB   10:00           644
  file2.docx   5 MB    11:30           644
  file3.jpg    3 MB    12:10           644

------------------------------------------------------------------------

# Step 2 -- Scan the Destination

Suppose the destination already contains:

``` text
/project

file1.pdf
file2.docx
```

DataSync retrieves the destination metadata.

------------------------------------------------------------------------

# Step 3 -- Compare Source and Destination

  File         Source   Destination   Action
  ------------ -------- ------------- --------
  file1.pdf    Same     Same          Skip
  file2.docx   Same     Same          Skip
  file3.jpg    Exists   Missing       Upload

Only **file3.jpg** is transferred.

------------------------------------------------------------------------

# What Happens When a File Changes?

Yesterday:

``` text
report.xlsx
Size: 20 MB
Modified: 09:00
```

Today:

``` text
report.xlsx
Size: 23 MB
Modified: 14:20
```

Comparison:

  Attribute       Source   Destination
  --------------- -------- -------------
  Name            Same     Same
  Size            23 MB    20 MB
  Modified Time   14:20    09:00

Since the metadata differs, DataSync transfers the updated file.

------------------------------------------------------------------------

# Does DataSync Keep a Database?

**No.**

Each task performs a fresh comparison.

``` text
Task Starts
    │
    ▼
Scan Source
    │
    ▼
Scan Destination
    │
    ▼
Compare
    │
    ▼
Transfer Only Differences
```

------------------------------------------------------------------------

# Comparison Methods

## Metadata Comparison (Default)

DataSync compares:

-   File name/path
-   File size
-   Last modified timestamp
-   Permissions
-   Ownership (where applicable)

This is fast and efficient.

------------------------------------------------------------------------

## Checksum Verification

DataSync can also verify data integrity using checksums.

``` text
File
 │
 ▼
Generate Checksum
 │
 ▼
Compare with Destination
 │
 ├── Match → Success
 └── Mismatch → Report/Transfer
```

Checksums are primarily used for validation after transfer.

------------------------------------------------------------------------

# Incremental Transfer Example

## First Run

Source:

``` text
A
B
C
D
```

Destination:

``` text
(empty)
```

Transferred:

``` text
A
B
C
D
```

------------------------------------------------------------------------

## Second Run

Source:

``` text
A
B
C
D
E
```

Destination:

``` text
A
B
C
D
```

Transferred:

``` text
E
```

------------------------------------------------------------------------

## Third Run

Source:

``` text
A
B
C (updated)
D
E
```

Destination:

``` text
A
B
C (old)
D
E
```

Transferred:

``` text
C
```

------------------------------------------------------------------------

# Deleted Files

Source:

``` text
A
B
```

Destination:

``` text
A
B
C
```

If the task is configured to delete destination files that no longer
exist in the source, DataSync deletes **C**. Otherwise, it leaves **C**
untouched.

------------------------------------------------------------------------

# Large Dataset Example

Imagine:

-   10 million files
-   50 TB total data

If only one file changes:

``` text
invoice_983221.pdf
```

The next task scans both locations, detects the single changed file, and
transfers **only that file**, not the entire dataset.

------------------------------------------------------------------------

# Internal Workflow

``` text
        Source Storage
              │
              ▼
      Read File Metadata
              │
              ▼
 Read Destination Metadata
              │
              ▼
     Compare Attributes
              │
      ┌───────┴────────┐
      │                │
  Same Metadata   Different Metadata
      │                │
     Skip         Queue for Transfer
                       │
                       ▼
                Copy to Destination
                       │
                       ▼
             Verify Data Integrity
```

------------------------------------------------------------------------

# Key Takeaways

-   DataSync does **not** maintain a permanent transfer database.
-   Every task scans both source and destination.
-   It compares metadata to determine what has changed.
-   Only new or modified files are transferred.
-   Checksum verification can validate transfer integrity.
-   This makes DataSync highly efficient for recurring synchronization
    tasks.

------------------------------------------------------------------------

# Interview Answer

> AWS DataSync determines what to transfer by comparing the source and
> destination during every task execution. It primarily compares
> metadata such as file path, file size, and last modified timestamp,
> and can optionally verify integrity with checksums. It does not keep a
> permanent record of previously transferred files. Instead, every run
> performs a fresh comparison and transfers only new or changed data.
