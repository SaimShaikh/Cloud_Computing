<img width="1102" height="372" alt="image" src="https://github.com/user-attachments/assets/3981e91d-0ec5-489a-8cd4-048d8303ded8" />

---

An S3 Lifecycle is a set of rules and actions that help automate the management of objects in a bucket.
It allows you to transition objects to cheaper storage classes or expire (delete) them after a certain period of time.

Key Features

- Written as rules and applied to a bucket or specific prefixes/tags.
- Works with versioned and non-versioned buckets.
- Automates cost optimization and cleanup without manual intervention.


### Common Lifecycle Actions
- Transition Actions – Move objects to other storage classes:
   - After 30 days → Move to S3 Standard-IA (Infrequent Access)
   - After 60 days → Move to S3 Glacier for archival
- Expiration Actions – Delete objects after a certain time.
- Non-Current Version Expiration – Delete older versions in versioned buckets.
- Abort Incomplete Multipart Uploads – Automatically remove incomplete uploads to save costs.


### Benefits

- Cost Optimization – Store infrequently accessed data in cheaper tiers.
- Automation – No need for manual cleanup or moving data.
- Compliance – Retain data for a fixed duration, then auto-delete it.
- Better Management – Helps manage storage growth effectively.

## Sample Lifecycle Rule (JSON)

```bash
{
  "Rules": [
    {
      "ID": "MoveToGlacier",
      "Status": "Enabled",
      "Prefix": "logs/",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 365
      }
    }
  ]
}
```
