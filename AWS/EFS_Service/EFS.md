<img width="1239" height="749" alt="Screenshot 2025-08-09 at 5 45 16 PM" src="https://github.com/user-attachments/assets/eac81ed4-1c4a-4ef3-9a4d-4d0ce3b8b3f0" />


# Understanding Amazon Elastic File System (EFS) with NFS

## What is EFS (NFS)?

**Amazon Elastic File System (EFS)** is a scalable, cloud-based file storage service from AWS. It uses the Network File System (NFS) protocol, allowing you to mount EFS on Linux-based EC2 instances and other compatible clients. EFS is designed for shared file access in the cloud, making it ideal for scenarios where multiple servers need access to the same data.

## Scenario: Shared Storage for a Web Application

Consider a web application deployed for high availability. The application’s servers (EC2 instances) sit behind a load balancer, and all servers need access to the same files—like user uploads, web assets, and logs.

**Without EFS:**  
You must manually synchronize files between servers or implement complex replication solutions, which is hard to manage as you add or remove servers.

**With EFS (NFS):**
- Create an EFS file system in AWS.
- Mount the EFS on all EC2 instances using NFS.
- All servers read and write to a single, shared data location.
- Scaling is easy—just attach new instances to the same EFS.

## Benefits of Using EFS (NFS)

- **Scalable:** Automatically grows/shrinks as files are added or removed. No manual resizing needed.
- **Highly Available:** Data is redundantly stored across multiple availability zones.
- **Concurrent Access:** Multiple servers can access and modify data at the same time with strong consistency.
- **Fully Managed:** AWS handles patching, maintenance, and backups.
- **POSIX-Compliant:** Works seamlessly with Linux permissions and most applications.
- **Secure:** Supports encryption at rest and in transit, with fine-grained access control.
- **Elastic & Cost-Efficient:** Pay only for what you use—no need to pre-provision capacity.

## Example Use Cases and Benefits

| Scenario                               | EFS Solution                   | Key Benefit                                   |
|-----------------------------------------|--------------------------------|------------------------------------------------|
| Web application with multiple servers   | Mount EFS to all servers       | Shared access & simple scaling                 |
| Big data processing (e.g., analytics)   | Shared workspace on EFS        | Central data access, easy collaboration        |
| File backup & restore                   | EFS + AWS Backup integration   | Automated, reliable backups                    |
| CI/CD pipelines / DevOps workflows      | Temporary files on EFS         | Persist artifacts through pipeline stages      |

## Summary

Amazon EFS with NFS is perfect for applications that need scalable, shared, and reliable storage across many compute resources in AWS. It minimizes operational overhead and complexity so you can focus on building powerful, cloud-native solutions.

## Setting Up Amazon EFS: Practical Steps (Particle)

### Requirements
1. **Two EC2 Instances**  
   For this example, we'll use two EC2 instances running **Amazon Linux**.
   
3. **Security Groups**  
   Port No 22 , 2049 , 80
   
5. **Amazon EFS File System**  
   An existing Amazon EFS file system created in your AWS account.
   
7. **EFS Package and EFS Directory with Full Permission You need to do this on Server**
   
   
   ## Images

   ### 1. Create Security Group for Server Attachment
   <img width="1644" height="925" alt="security " src="https://github.com/user-attachments/assets/ef55e731-43f4-4232-bf30-ef9a7c8bc32b" />

   ### 2. Create 2 EC2 Server and Attach Security Groups which you created  
   <img width="1369" height="209" alt="Screenshot 2025-07-31 at 11 05 35 PM" src="https://github.com/user-attachments/assets/4f2dfd68-6a79-4a97-a559-8996890083e6" />
   
   ### 3. Install EFS Package In both servers
   ``` bash
   yum install amazon-efs-utils -y
   mkdir efs
   chmod 777 efs
   ```
   ### 4. Create EBS (NFS)
   # Go to Search Bar and Type EBS 
   <img width="912" height="179" alt="Screenshot 2025-07-31 at 11 21 37 PM" src="https://github.com/user-attachments/assets/662cd029-f603-409f-8df6-52b572e9ae8c" />
   
   ### 5.Click on Create 
   <img width="1649" height="964" alt="Screenshot 2025-07-31 at 11 23 45 PM" src="https://github.com/user-attachments/assets/8752af7b-fa1a-42cc-bfc5-6a05bb548de4" />

   ### 6.Click on Customise
   
   <img width="1615" height="1009" alt="Cus" src="https://github.com/user-attachments/assets/6e3c8dc8-282f-4e59-88f2-0a498bac7d23" />

   ### 7. Fill and Select and Scroll down and Click Next
   <img width="1582" height="906" alt="name" src="https://github.com/user-attachments/assets/fa81b1e1-6d44-4060-9523-3f41a4778f5a" />

   ### 8. In Mount Target Assign Your Security Group which you Created to all Availability zone
   <img width="1597" height="967" alt="assign " src="https://github.com/user-attachments/assets/05c989c6-b36f-4aa4-afe1-7b9a4d7d0d69" />

   ### 9. Create And Wait for Status (Available) then Click On EFS Name 
   <img width="1680" height="1050" alt="wait" src="https://github.com/user-attachments/assets/e9b8bc7f-12c5-4e9c-9e89-1c5a8c4aa82c" />
  ### 10. Click Network and Check Status all Should be Available 
  
   <img width="1612" height="974" alt="n" src="https://github.com/user-attachments/assets/db3b3c0f-164f-413d-b599-80944fdfafbe" />
   
   ### 11. Scroll up and Click Attach 
   
  <img width="1680" height="1050" alt="at" src="https://github.com/user-attachments/assets/88dfa7c8-c083-425f-ae48-4815028589b6" />
  
  ### 12. Click on Mount via DNS and Copy command to paste on both EC2 Servers
  
   <img width="1620" height="991" alt="copy " src="https://github.com/user-attachments/assets/97a7bb87-9cc6-43d8-9919-93e1cabddb06" />
   
   ### 13. After That you can create any file or directory inside efc directory it will visible on both side  
   
   <img width="1652" height="990" alt="fin" src="https://github.com/user-attachments/assets/a180bf96-c705-40d9-a238-f55a9eb5670b" />

# Conclusion

You have successfully set up Amazon Elastic File System (EFS) with two Amazon Linux EC2 instances, enabling shared, scalable, and highly available file storage using NFS. This setup is ideal for distributed applications, scaling web servers, or any scenario requiring synchronized file access across hosts.



   

   
   
