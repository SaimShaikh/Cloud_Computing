# SSM Agent Installation and Login to Private EC2 Using AWS Systems Manager (SSM)

## Objective

In this lab, we will:

* Create a custom VPC
* Create Public and Private Subnets
* Configure Internet Gateway
* Configure Route Tables
* Create Security Groups
* Create IAM Role for Systems Manager
* Launch a Private EC2 Instance
* Configure VPC Interface Endpoints
* Register EC2 with AWS Systems Manager
* Connect to Private EC2 using Session Manager

### Benefits

* No Public IP required
* No SSH Port (22) required
* No Bastion Host required
* Secure access to private servers
* Production-ready architecture

---

# Architecture Overview

```text
AWS Systems Manager
        |
-------------------------------------------------
|                 |                           |
SSM          SSMMessages                 EC2Messages
|                 |                           |
-------------------------------------------------
            VPC Interface Endpoints
                    |
            Private Subnet
                    |
           Private EC2 Instance
                    |
                IAM Role
                    |
AmazonSSMManagedInstanceCore
                    |
                SSM Agent
                    |
                   VPC
          ------------------
          |                |
     Public Subnet   Private Subnet
          |
    Internet Gateway
```

---

# Step 1: Create VPC

Navigate to:

```text
VPC → Your VPCs → Create VPC
```

Configuration:

| Parameter | Value       |
| --------- | ----------- |
| Name      | ssm-vpc     |
| IPv4 CIDR | 10.0.0.0/16 |

Click:

```text
Create VPC
```

---

# Step 2: Enable DNS Settings

Navigate:

```text
VPC → Your VPCs
```

Select:

```text
ssm-vpc
```

Click:

```text
Actions → Edit VPC Settings
```

Enable:

```text
Enable DNS Resolution
Enable DNS Hostnames
```

Save Changes.

---

# Step 3: Create Public Subnet

Navigate:

```text
VPC → Subnets → Create Subnet
```

Configuration:

| Parameter | Value         |
| --------- | ------------- |
| Name      | public-subnet |
| VPC       | ssm-vpc       |
| AZ        | ap-south-1a   |
| CIDR      | 10.0.1.0/24   |

Create Subnet.

---

# Step 4: Create Private Subnet

Navigate:

```text
VPC → Subnets → Create Subnet
```

Configuration:

| Parameter | Value          |
| --------- | -------------- |
| Name      | private-subnet |
| VPC       | ssm-vpc        |
| AZ        | ap-south-1a    |
| CIDR      | 10.0.2.0/24    |

Create Subnet.

---

# Step 5: Create Internet Gateway

Navigate:

```text
VPC → Internet Gateways
```

Create:

| Parameter | Value   |
| --------- | ------- |
| Name      | ssm-igw |

Click Create.

Attach to:

```text
ssm-vpc
```

---

# Step 6: Create Public Route Table

Navigate:

```text
VPC → Route Tables
```

Create Route Table:

| Parameter | Value     |
| --------- | --------- |
| Name      | public-rt |
| VPC       | ssm-vpc   |

---

# Step 7: Configure Public Route

Open:

```text
public-rt
```

Routes:

```text
Destination : 0.0.0.0/0
Target      : Internet Gateway
```

Select:

```text
ssm-igw
```

Save.

---

# Step 8: Associate Public Subnet

Open:

```text
public-rt
```

Select:

```text
Subnet Associations
```

Associate:

```text
public-subnet
```

Save.

---

# Step 9: Create Private Route Table

Navigate:

```text
VPC → Route Tables
```

Create:

| Parameter | Value      |
| --------- | ---------- |
| Name      | private-rt |
| VPC       | ssm-vpc    |

---

# Step 10: Associate Private Subnet

Open:

```text
private-rt
```

Subnet Associations:

```text
private-subnet
```

Save.

Routes should contain only:

```text
10.0.0.0/16 → local
```

No Internet Route.

---

# Step 11: Create EC2 Security Group

Navigate:

```text
EC2 → Security Groups
```

Create:

| Parameter | Value      |
| --------- | ---------- |
| Name      | ssm-ec2-sg |
| VPC       | ssm-vpc    |

Inbound Rules:

```text
None
```

Outbound:

```text
All Traffic
```

Create Security Group.

---

# Step 12: Create VPC Endpoint Security Group

Navigate:

```text
EC2 → Security Groups
```

Create:

| Parameter | Value   |
| --------- | ------- |
| Name      | vpce-sg |
| VPC       | ssm-vpc |

Inbound Rule:

| Type  | Port | Source      |
| ----- | ---- | ----------- |
| HTTPS | 443  | 10.0.0.0/16 |

Outbound:

```text
All Traffic
```

Create Security Group.

---

# Step 13: Create IAM Role

Navigate:

```text
IAM → Roles → Create Role
```

Select:

```text
AWS Service
```

Use Case:

```text
EC2
```

Click Next.

Attach Policy:

```text
AmazonSSMManagedInstanceCore
```

Role Name:

```text
SSM-Role
```

Create Role.

---

# Step 14: Launch Private EC2

Navigate:

```text
EC2 → Launch Instance
```

Configuration:

| Parameter      | Value             |
| -------------- | ----------------- |
| Name           | private-server    |
| AMI            | Amazon Linux 2023 |
| Instance Type  | t3.micro          |
| VPC            | ssm-vpc           |
| Subnet         | private-subnet    |
| Public IP      | Disabled          |
| Security Group | ssm-ec2-sg        |
| IAM Role       | SSM-Role          |

Launch Instance.

---

# Step 15: Create SSM Interface Endpoint

Navigate:

```text
VPC → Endpoints → Create Endpoint
```

Service:

```text
com.amazonaws.ap-south-1.ssm
```

Configuration:

| Parameter      | Value          |
| -------------- | -------------- |
| VPC            | ssm-vpc        |
| Subnet         | private-subnet |
| Security Group | vpce-sg        |
| Private DNS    | Enabled        |
| Policy         | Full Access    |

Create Endpoint.

Wait until:

```text
Available
```

---

# Step 16: Create SSMMessages Endpoint

Create another endpoint.

Service:

```text
com.amazonaws.ap-south-1.ssmmessages
```

Same configuration:

```text
VPC: ssm-vpc
Subnet: private-subnet
Security Group: vpce-sg
Private DNS: Enabled
Policy: Full Access
```

Create Endpoint.

Wait for:

```text
Available
```

---

# Step 17: Create EC2Messages Endpoint

Create another endpoint.

Service:

```text
com.amazonaws.ap-south-1.ec2messages
```

Use same configuration.

Create Endpoint.

Wait for:

```text
Available
```

---

# Step 18: Verify Managed Node

Navigate:

```text
AWS Systems Manager
→ Fleet Manager
→ Managed Nodes
```

Expected Result:

```text
private-server
```

Status:

```text
Running
```

Ping Status:

```text
Online
```

---

# Step 19: Connect Using Session Manager

Navigate:

```text
EC2
→ Instances
→ private-server
→ Connect
→ Session Manager
→ Connect
```

Expected:

```bash
sh-5.2$
```

or

```bash
ec2-user@ip-10-0-2-x
```

---

# Verification Commands

Check current user:

```bash
whoami
```

Check hostname:

```bash
hostname
```

Check IP address:

```bash
ip addr
```

Check IAM Role access:

```bash
aws sts get-caller-identity
```

---

# Expected Outcome

You have successfully:

* Created a custom VPC
* Created Public and Private Subnets
* Configured Route Tables
* Attached Internet Gateway
* Created Security Groups
* Created IAM Role
* Attached AmazonSSMManagedInstanceCore Policy
* Launched Private EC2
* Created 3 VPC Interface Endpoints
* Registered EC2 with Systems Manager
* Connected to Private EC2 without SSH
* Connected without Public IP
* Connected without Bastion Host

---

# Cleanup

Delete resources in the following order:

1. Terminate EC2 Instance
2. Delete VPC Endpoints
3. Delete Security Groups
4. Delete Route Tables
5. Detach and Delete Internet Gateway
6. Delete Subnets
7. Delete VPC
8. Delete IAM Role (Optional)

This avoids unnecessary AWS charges.

