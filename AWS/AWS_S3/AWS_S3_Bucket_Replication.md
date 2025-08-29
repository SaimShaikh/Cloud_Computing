# Bucket Replication 

S3 replication is a feature that allows objects in one bucket (the source) to be automatically copied to another bucket (the destination) in the same or different AWS regions.


Types of Replication

- Cross-Region Replication (CRR) – Replicates data to a bucket in a different AWS region for disaster recovery, latency optimization, or compliance.

- Same-Region Replication (SRR) – Replicates data within the same region for use cases like logging, compliance, or maintaining a secondary copy.

Key Points
- Requires versioning to be enabled on both source and destination buckets.
- Replication is asynchronous, so there might be a small delay.
- Supports replicating new objects, object metadata, ACLs, and tags.
- Can filter replication by prefix or tags.
- Works with KMS encryption, but needs additional permissions.

Benefits

- Disaster Recovery – Data is safe even if one region goes down.
- Compliance – Storing data in specific regions to meet legal or corporate requirements.
- Low-Latency Access – Keeping a copy closer to users in different geographies.
- Backup & Redundancy – Maintaining a secure secondary copy.


### Pre-requites 
- 2 Buckets (One for Source and one for destination )
- Same or Different Region

Step 1. Create 2 Bucktes

---
<img width="2195" height="646" alt="image" src="https://github.com/user-attachments/assets/c5d5fa8c-108c-44e6-8e40-9a6dbf7ec605" />

---

Step 2. Go to your first (Original) Bucket store data and click on Management 
<img width="3315" height="1295" alt="image" src="https://github.com/user-attachments/assets/882e001c-72df-4ff7-84e4-e8357926d541" />

---
Step 3. Scroll Down find Replication rules click on Create replication rule
<img width="3289" height="626" alt="image" src="https://github.com/user-attachments/assets/cdbb5fac-7ad7-4552-bbdb-0950603b3be6" />

---

Step 4. Create replication rule
<img width="3331" height="897" alt="image" src="https://github.com/user-attachments/assets/0f2e55a7-88e7-41da-b421-fe81feb11113" />

---
Step 5. Source bucket 

<img width="3300" height="555" alt="image" src="https://github.com/user-attachments/assets/90f6c671-a3ed-4a40-8ca2-af9ef7ed1ec4" />

---

Step 6. Select Destination Bucket where you want to copy data 
<img width="3239" height="729" alt="image" src="https://github.com/user-attachments/assets/376c558d-e514-4d98-91a4-2fcc7004b3ed" />

- Select Destination bucket name
  
<img width="3231" height="894" alt="image" src="https://github.com/user-attachments/assets/91d5ed6e-9662-4ffa-b8ae-9e8b9e085e08" />

---

Step 7. IAM Role Click on Create new role
<img width="3289" height="396" alt="image" src="https://github.com/user-attachments/assets/52d0d382-0fe0-45f8-8f31-84c7612252de" />

---
Step 8. Replicate existing objects? Click on Yes, replicate existing objects. and Submit 
<img width="3329" height="1861" alt="image" src="https://github.com/user-attachments/assets/85199ae7-f8ac-46b2-9009-eda2daa948d1" />

---

Step 9. Create Batch Operations job
<img width="3352" height="1797" alt="image" src="https://github.com/user-attachments/assets/9a4c599b-c25e-4003-9379-23b0db4982c7" />

---
Step 10. Permission 
<img width="3299" height="769" alt="image" src="https://github.com/user-attachments/assets/615a6216-774d-44bf-a550-8276e3f76bbf" />

---
Step 11. Now you Successfully created batch operations and wait for some time  
<img width="3324" height="1206" alt="image" src="https://github.com/user-attachments/assets/538545c9-1300-458f-b204-1a7014e47790" />

---
Step 12. Check you newly created bucket (Destination) you will see all data are successfully copied form old bucket (source) 
<img width="2755" height="1267" alt="image" src="https://github.com/user-attachments/assets/dc0bcd3c-0090-4576-b8df-4e6e1e714d98" />

## if you add any data in old bucket s3 will automatically copy that file into new bucket 

> Download code https://github.com/SaimShaikh/Cloud_Computing/blob/main/AWS/AWS_S3/AWS_S3_Zip.md
