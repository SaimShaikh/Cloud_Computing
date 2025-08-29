# What is the bucket versioning ?
S3 bucket versioning is a feature that allows you to keep multiple versions of an object in the same bucket.
Whenever an object is modified or deleted, S3 stores the previous version instead of overwriting or permanently removing it.

or 
Bucket versioning in S3 allows us to maintain multiple versions of objects in the same bucket. It’s very helpful for recovering accidentally deleted files or restoring older versions of data. I’ve used it in a project where our logs and configuration files were versioned to prevent accidental loss and to meet audit compliance requirements.

### Key Points
- Disabled by default; must be explicitly enabled.
- Once enabled, every object has a unique version ID.
- Delete operations don’t permanently remove objects; they just add a delete marker.
- Works well with MFA Delete for additional security.

### Benefits

- Data Recovery – Restore accidentally deleted or overwritten objects.
- Change Tracking – Maintain a history of changes to objects.
- Backup & Compliance – Useful for businesses that require audit trails or strict data retention.

### My website with old version 

<img width="3261" height="1822" alt="image" src="https://github.com/user-attachments/assets/99ace37a-f9a3-4071-ad57-5d65f7d2c1e7" />

Step 1. Go to Buckets properties click on edit 
<img width="3300" height="1027" alt="image" src="https://github.com/user-attachments/assets/d81ba2c4-9415-469b-9e22-9237015126f3" />

Step 2. Click to Upload new file 
<img width="3332" height="1264" alt="image" src="https://github.com/user-attachments/assets/0e030c75-6041-4d3f-bb2a-724825704654" />


Step 3. Add new updated index.html file to see new version 
<img width="3279" height="748" alt="image" src="https://github.com/user-attachments/assets/f1fc5a5e-7d80-40d3-ad20-d311b7a378b8" />

I made small change in my index.html code to see versions 

--- 

<img width="561" height="100" alt="image" src="https://github.com/user-attachments/assets/5594a213-7ce0-433c-ae01-68e73a663968" />



Step 4. Check your Object list you can see a new version of index.html has been created
<img width="3321" height="1306" alt="image" src="https://github.com/user-attachments/assets/0b255462-13b2-4859-ab74-d834e2e8e1ae" />

# After that if you Delete new update index.html s3 automatically shifts back to old version


Step 5. Refresh you websute tab and see Output
<img width="3327" height="1916" alt="image" src="https://github.com/user-attachments/assets/cc3e3e48-3d13-45ca-9813-e74a5c4c61e8" />

- Download code https://github.com/SaimShaikh/Cloud_Computing/blob/main/AWS/AWS_S3/AWS_S3_Zip.md
