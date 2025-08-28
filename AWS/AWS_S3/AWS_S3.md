<img width="500" height="500" alt="image" src="https://github.com/user-attachments/assets/08aa3d7f-ca72-4991-850d-74a2049c710e" />

# AWS S3
## What is the s3 bucket
Amazon S3, or Simple Storage Service, is an object storage service provided by AWS. It allows us to store and retrieve any amount of data at any time from anywhere over the internet.

In S3, a bucket is basically the top-level container where we store objects, which can be files, images, backups, or any type of data. Each bucket has a unique name globally and is tied to a specific AWS region.

Buckets also allow us to manage data using features like versioning, lifecycle policies, encryption, access control, and logging. For example, I’ve used S3 buckets for hosting static websites, storing application logs, and even as a backend for backups.

The data in S3 is automatically replicated across multiple Availability Zones, which ensures durability and high availability — AWS guarantees 99.999999999% durability for objects.
