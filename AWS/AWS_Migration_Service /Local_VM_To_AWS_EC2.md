
# Local VM to AWS Migration

## Overview

Local VM to AWS Migration is the process of moving virtual machines running in an on-premises environment (VMware, Hyper-V, VirtualBox, KVM, etc.) to Amazon Web Services (AWS). This migration helps organizations reduce infrastructure management overhead, improve scalability, increase availability, and leverage AWS cloud services.

---

# Migration Type

This migration falls under:

* **Rehosting (Lift and Shift)** – Moving servers with minimal or no modifications.
* **Infrastructure Migration** – Migrating workloads from on-premises infrastructure to AWS Cloud.

In most cases, Local VM to AWS migration is considered a **Lift and Shift Migration**.

---

# Prerequisites

Before starting the migration, ensure the following requirements are met:

## Requirements to Perform this 

* AWS Account
* IAM User with required permissions
* VPC and Subnets
* Security Groups
* Sufficient EC2 quotas
* Migration Agent
* AWS Application Migration Service (MGN)
* S3 Bucket (if required)
* VM Box
* ubuntu-22.04.5-desktop-amd64 This ISO file or older 

---

# Migration Approaches
## 1. Setup the virtual Box with ubuntu-22.04.5-desktop-amd64 image and in this Machine i installed nginx and some text files .

<img width="2565" height="1604" alt="image" src="https://github.com/user-attachments/assets/852a9a18-a3ce-4113-a0b0-80acc662b8a7" />

---

<img width="2555" height="1594" alt="image" src="https://github.com/user-attachments/assets/3fb02071-bb0a-4acf-abe7-a5db71fd3ff3" />

---

## 2: Create IAM User

Navigate to:

```text
IAM → Users → Create User
```

Create user:

```text
mgn-migration-user
```

Attach permissions:

```text
AWSApplicationMigrationAgentInstallationPolicy
AWSApplicationMigrationFullAccess
```

Create Access Key:

```text
IAM User
→ Security Credentials
→ Create Access Key
```

Save:

```text
Access Key ID
Secret Access Key
```

---

## 3. VPC Creation 

1. Search in AWS: Amazon Virtual Private Cloud

2 Left side: 
- Your VPCs
- Click:
- Create VPC

Configure Setting Value
- Resources to create = VPC only
- Name tag = migration-vpc
- IPv4 CIDR = 10.0.0.0/16
- IPv6 = None
- Tenancy = Default

**Create VPC**

2. CREATE PUBLIC SUBNET

- Left side:
   - Subnets
- Click:
  - Create subnet


Configure Setting Value
- VPC = migration-vpc
- Name = public-subnet
- AZ = ap-south-1a
- CIDR = 10.0.1.0/24

**Create subnet**

3. CREATE INTERNET GATEWAY
- Left side:
   - Internet Gateways
- Click:
   - Create internet gateway

- Configure Setting Value
  - Name = migration-igw

**Create internet gateway**

4. ATTACH IGW TO VPC
Select: migration-igw > Click  >Actions Choose > Attach to VPC

Select > migration-vpc > Click > Attach internet gateway

5. CREATE ROUTE TABLE
- Left side:
    - Route Tables
- Click:
    - Create route table

Configure Setting Value
- Name = public-route-table
- VPC = migration-vpc

**Create route table**

6. ADD INTERNET ROUTE

Select > public-route-table
Go To > Routes tab > Click > Edit routes

Add Route 
- Destination =  0.0.0.0/0
- Target = Internet Gateway

Choose: > migration-igw > save 

7. ASSOCIATE SUBNET

Go > Subnet Associations 
Click: > Edit subnet associations
Select > public-subnet

**Save**

8. ENABLE PUBLIC IP

Go > Subnets

Select > public-subnet

Actions > Choose  > Edit subnet settings

Enable > Auto-assign public IPv4

**Save**


9. CREATE SECURITY GROUP

Create one Security Group with port 22 ,1500 , 80 

---


## 4. AWS Application Migration Service 

AWS MGN continuously replicates source servers to AWS and minimizes downtime during cutover.


### Benefits

* Near-zero downtime
* Automated replication
* Supports large-scale migrations
* Cost-effective

Initialize the service.

AWS automatically creates:

* Replication Templates
* Service Roles
* Replication Infrastructure

### Important Note About Instance Types

By default AWS may automatically select:

- c5.large
- m5.large
- c5.xlarge

based on workload analysis.

To prevent this:

MGN

→ Launch Settings
→ Launch Template

Disable:

Automatic Right Sizing

Manually select:

- t3.micro
- t3.small
- t3.medium

for testing environments.

---

## Step 5: Configurations for Installing  Replication Agent 

<img width="1365" height="912" alt="Screenshot 2026-05-24 at 4 50 26 PM" src="https://github.com/user-attachments/assets/8abe7551-422e-4bc1-8611-d8706fadfa62" />


- after that  Copy the Commands and Paste in your Local Machine 

```bash
wget https://aws-application-migration-service-agent-url
chmod +x aws-replication-installer-init 
```

Successful installation message:

```text
The AWS Replication Agent was successfully installed.
```

### Windows

* Download the installer.
* Run as Administrator.
* Provide AWS credentials when prompted.

---

# Step 6: Verify Source Server

Navigate to:

```text
AWS MGN
→ Source Servers
```

Verify:

```text
Server discovered
```

Status initially:

```text
Initial Sync
```

- If Python Missing

```sudo apt install python3 -y```

## Step 4: Start Replication Monitoring (It Will Take around 78 - 79 min )

<img width="3023" height="1626" alt="image" src="https://github.com/user-attachments/assets/23777d49-0ade-424e-bc34-5437f81f296f" />

---

<img width="3023" height="1626" alt="image" src="https://github.com/user-attachments/assets/c715ce3b-cd8d-4e9e-9850-a51989fd6b3f" />


* Agent begins replicating data to AWS.
* Monitor replication status from AWS MGN Console.
* Wait until the server reaches a healthy state.

---

## Step 5: Launch Test Instance

* Launch a test EC2 instance.
* Validate:

  * Application functionality
  * Network connectivity
  * Storage accessibility
  * Performance

---

## Step 6: Perform Cutover

* Schedule migration window.
* Stop source application.
* Launch Cutover Instance.
* Validate production traffic.

---

## Step 7: Post-Migration Validation

Verify:

* Application health
* CPU and memory utilization
* Storage performance
* Security configuration
* Monitoring and logging

---

# Architecture Flow
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/6c41a9a3-41a5-4944-88da-491ff46bd77e" />


---

# Important Considerations

## Network

* Verify bandwidth availability.
* Consider AWS Direct Connect or VPN for large migrations.

## Security

* Encrypt data during transfer.
* Follow least-privilege IAM principles.
* Validate security groups and firewall rules.

## Licensing

* Check Windows licensing requirements.
* Verify third-party software licenses.

## Storage

* Ensure sufficient EBS storage.
* Select appropriate volume type:

  * gp3
  * io2
  * st1

## Downtime Planning

* Perform migration during maintenance windows.
* Test rollback procedures.

---

# Common Challenges

| Challenge                     | Solution                             |
| ----------------------------- | ------------------------------------ |
| Slow replication              | Increase bandwidth                   |
| Application dependency issues | Dependency mapping before migration  |
| Large database migration      | Separate database migration strategy |
| Driver incompatibility        | Install AWS drivers and agents       |
| Network latency               | Use Direct Connect or VPN            |

---

# Best Practices

* Always perform a test migration first.
* Maintain backups before migration.
* Validate application functionality after migration.
* Enable CloudWatch monitoring.
* Document rollback procedures.
* Use AWS MGN for production workloads.
* Conduct security and compliance checks.

---

# Rollback Plan

If migration fails:

1. Keep source VM running until validation is complete.
2. Redirect traffic back to on-premises VM.
3. Investigate issues.
4. Fix and reattempt migration.

Never decommission the source server until migration validation is completed successfully.

---

# Conclusion

Local VM to AWS Migration enables organizations to modernize infrastructure and leverage AWS cloud capabilities. AWS Application Migration Service (MGN) is the preferred approach because it provides continuous replication, minimal downtime, scalability, and simplified migration management.
