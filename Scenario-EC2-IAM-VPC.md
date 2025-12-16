# Scenario Based Interview Questions: EC2, IAM, and VPC

> Real-world scenarios with practical answers. Simple language, no fluff.

---

## VPC Architecture Scenarios

### Q1. Design a VPC architecture for a 2-tier application that is highly available and scalable.

**Scenario:** You need to build a web application that can handle traffic spikes and survive data center failures.

**Answer:**

I would design the VPC architecture like this:

**Network Design:**
- Create 2 public subnets (one in each availability zone) for the load balancer and web servers.
- Create 2 private subnets (one in each availability zone) for application servers.
- Create 2 private subnets (one in each availability zone) for databases.

**Internet Connectivity:**
- Attach an Internet Gateway to the VPC for public subnet access.
- Use NAT Gateways in public subnets so private instances can reach the internet for updates.

**Traffic Flow:**
```
Users → Internet Gateway → Application Load Balancer (in public subnet)
                         ↓
                    EC2 Instances (in public subnet)
                         ↓
                    Private Subnets (application servers)
                         ↓
                    RDS Database (in private subnet)
```

**High Availability:**
- Distribute subnets across 2+ Availability Zones.
- Use Application Load Balancer (ALB) to distribute traffic.
- Auto-scaling groups for EC2 instances (scale 2-10 instances based on load).
- Multi-AZ RDS database with automatic failover.

**Security:**
- Security groups: Allow HTTP/HTTPS on web tier, restrict database access to app tier.
- Network ACLs: Additional filtering at subnet level.
- Bastion host in public subnet for secure SSH access to private instances.

---

### Q2. Your organization has a VPC with multiple subnets. You want to restrict outbound internet access for resources in one subnet, but allow outbound internet access for resources in another subnet. How would you achieve this?

**Scenario:** You have sensitive databases that should not reach the internet, but web servers need to download updates.

**Answer:**

**For the restricted subnet (no outbound internet):**
- Remove the default route (0.0.0.0/0) from the route table.
- Don't attach a NAT Gateway or Internet Gateway.
- Instead, create specific routes only to resources they need (e.g., only to application servers).

Example Route Table for isolated subnet:
```
Destination        Target
10.0.0.0/16       Local (VPC)
(nothing else)
```

**For the subnet with outbound internet:**
- Keep the default route pointing to Internet Gateway OR NAT Gateway.

Example Route Table for web subnet:
```
Destination        Target
10.0.0.0/16       Local (VPC)
0.0.0.0/0         Internet Gateway
```

**Key difference:**
- **No internet route** = no outbound internet access.
- **Route to IGW** = can reach internet.
- **Route to NAT Gateway** = can initiate outbound connections but no inbound.

**In practice:** 
Databases stay isolated; web servers can update packages from internet package repositories.

---

### Q3. You have a VPC with a public subnet and a private subnet. Instances in the private subnet need to access the internet for software updates. How would you allow internet access for instances in the private subnet?

**Scenario:** Your app servers in private subnet need to download libraries/patches from the internet, but you don't want them directly exposed to the internet.

**Answer:**

Use a **NAT Gateway** (or NAT instance, but NAT Gateway is preferred).

**Setup:**

1. **Create NAT Gateway in Public Subnet**
   - Place in the public subnet so it has internet access.
   - Associate with an Elastic IP (static public IP).

2. **Update Private Subnet Route Table**
   - Add route: `0.0.0.0/0 → NAT Gateway`
   - Now traffic from private instances goes through NAT Gateway.

**How it works:**
```
Private Instance (10.0.2.5)
       ↓
   NAT Gateway (10.0.1.10)
       ↓
Internet Gateway
       ↓
Internet (appears to come from NAT Gateway's Elastic IP)
```

**Key points:**
- Private instances **initiate** outbound connections (like downloading packages).
- NAT Gateway translates private IP to the Elastic IP.
- Internet server responds to the Elastic IP.
- NAT Gateway forwards response back to private instance.
- **Inbound traffic cannot be initiated from internet** – only responses to outbound requests.

**Pricing note:** NAT Gateways cost money; NAT instances don't (but require manual management).

---

### Q4. You have launched EC2 instances in your VPC, and you want them to communicate with each other using private IP addresses. What steps would you take to enable this communication?

**Scenario:** You have web servers and database servers in different subnets. They need to talk to each other privately.

**Answer:**

**Good news:** By default, instances in the same VPC can communicate using private IPs.

**Steps to ensure communication works:**

1. **Same VPC**
   - Both instances must be in the same VPC (10.0.0.0/16).

2. **Network connectivity**
   - If in same subnet: Direct communication (no special setup).
   - If in different subnets: Route tables must have routes between them. By default, "Local" route allows this.

3. **Configure Security Groups**
   This is the critical part:
   
   **Source instance security group:**
   - Allow outbound on port needed (e.g., 3306 for MySQL).
   
   **Target instance security group:**
   - Allow inbound on that same port from source security group.

Example:
```
Web Server Security Group (sg-web):
  Outbound: 3306/TCP → sg-database

Database Security Group (sg-database):
  Inbound: 3306/TCP from sg-web
```

4. **Check Network ACLs** (if using restrictive NACLs)
   - Ensure inbound and outbound rules allow the traffic.

**Key principle:** 
By default, all instances in a VPC can reach each other. Security Groups control who actually can.

---

### Q5. You want to implement strict network access control for your VPC resources. How would you achieve this?

**Scenario:** Your organization has strict security requirements. You want multiple layers of network security.

**Answer:**

Use a **combination of Network ACLs and Security Groups** (defense in depth).

**At Subnet Level – Network ACLs:**

NACLs are stateless rules that filter traffic at the subnet boundary.

Example NACL for private subnet:
```
Rule 100: Allow inbound on port 22 from bastion (10.0.1.0/24)
Rule 110: Allow inbound on port 3306 from app servers (10.0.2.0/24)
Rule 120: Allow all outbound traffic
Rule 130: Deny all else
```

**At Instance Level – Security Groups:**

Security groups are stateful; allow specific traffic to instances.

Example Security Group for database instance:
```
Inbound Rules:
  - 3306/TCP from App Server SG (sg-app)
  - 22/TCP from Bastion SG (sg-bastion) [for admin]

Outbound Rules:
  - All traffic (default)
```

**Layered Approach:**

```
Internet
  ↓
[NACL – Stateless filtering at subnet boundary]
  ↓
Instance
  ↓
[Security Group – Stateful filtering at instance level]
  ↓
Application
```

**Benefits:**
- **NACL** catches broad attacks at subnet level.
- **Security Group** provides fine-grained control per instance.
- If one is misconfigured, the other provides backup protection.

**Best practice:** Use both; don't skip NACLs thinking security groups are enough.

---

### Q6. Your organization requires an isolated environment within the VPC for running sensitive workloads. How would you set up this isolated environment?

**Scenario:** You have confidential data or proprietary algorithms running. You want them completely isolated from the internet.

**Answer:**

Create an **Isolated Subnet** (also called a Private Isolated Subnet).

**Setup:**

1. **Create a Subnet with No Internet Routes**
   ```
   Subnet CIDR: 10.0.3.0/24
   Route Table:
     10.0.0.0/16 → Local (VPC traffic only)
     (No route to Internet Gateway or NAT Gateway)
   ```

2. **Place Sensitive Resources Here**
   - Sensitive databases
   - Machine learning models
   - Confidential data processing servers

3. **Network Isolation**
   - No inbound from internet (no IGW or NAT).
   - No outbound to internet.
   - Can only communicate with other instances in the VPC.

4. **If They Need Internet Access (for updates):**
   - Use VPC Endpoints instead of NAT Gateway.
   - VPC endpoints allow private access to AWS services (S3, DynamoDB, etc.).
   - No internet exposure.

5. **Administrative Access (if needed):**
   - Use AWS Systems Manager Session Manager (no SSH needed).
   - Or bastion host in public subnet → jump to private subnet → jump to isolated subnet.

**Example Architecture:**
```
Isolated Subnet (10.0.3.0/24)
  - Sensitive Database
  - No internet connectivity
  - Only accessible from app servers in private subnet
  - Accessed via VPC Endpoints for S3/DynamoDB
```

**Security Benefit:**
Even if a malicious actor compromises an EC2 instance, they cannot reach the internet or download tools/malware.

---

### Q7. Your application needs to access AWS services, such as S3, securely within your VPC. How would you achieve this?

**Scenario:** Your EC2 instances need to read/write from S3, but you don't want the traffic going over the public internet.

**Answer:**

Use **VPC Endpoints**.

**Without VPC Endpoint:**
```
EC2 in private subnet → NAT Gateway → IGW → Internet → S3
(Traffic crosses public internet)
```

**With VPC Endpoint:**
```
EC2 in private subnet → VPC Endpoint → S3
(Traffic stays within AWS network)
```

**Setup VPC Endpoint for S3:**

1. **Go to VPC Dashboard → Endpoints**
2. **Create Endpoint**
   - Service: `com.amazonaws.us-east-1.s3`
   - VPC: Select your VPC
   - Route Tables: Select subnets that need S3 access

3. **Configure Security**
   - Attach a policy allowing access (usually allow S3 actions on specific buckets).

4. **Update Route Table**
   - Route for S3 traffic automatically points to the endpoint.

**No instance configuration needed** – traffic is automatically routed through the endpoint.

**Benefits:**
- No NAT Gateway charges for S3 traffic.
- Faster (direct connection within AWS network).
- More secure (doesn't cross internet).
- Can restrict S3 access to specific buckets.

**Common VPC Endpoints:**
- **S3** (object storage)
- **DynamoDB** (NoSQL database)
- **API Gateway** (custom APIs)
- **SNS, SQS, CloudWatch** (managed services)

---

### Q8. What is the difference between NACLs and Security Groups? Explain with a use case.

**Scenario:** You want to design security for a multi-tier application with strict controls.

**Answer:**

| Aspect | Network ACL | Security Group |
|--------|-------------|---|
| **Scope** | Subnet-level | Instance-level |
| **Stateful?** | No (stateless) | Yes (stateful) |
| **Rules Type** | Allow and Deny | Allow only (no explicit deny) |
| **Evaluation** | First match wins | All rules evaluated |
| **Rule Order** | Important (processed top-to-bottom) | Not important |
| **Performance Impact** | Low | Low |

**Detailed Comparison:**

**Network ACL (Stateless):**
- Filter traffic at **subnet boundary**.
- Apply to all instances in the subnet.
- Must explicitly define both inbound AND outbound.

Example: Deny all traffic except port 443 (HTTPS)
```
Inbound:
  Rule 100: Allow 443/TCP
  Rule 200: Deny all else

Outbound:
  Rule 100: Allow 443/TCP
  Rule 200: Deny all else
```

**Security Group (Stateful):**
- Filter traffic at **instance level**.
- Apply only to instances it's attached to.
- If you allow inbound, outbound response is automatic.

Example: Allow HTTPS from internet
```
Inbound:
  Allow 443/TCP from 0.0.0.0/0

Outbound:
  (Automatic response allowed; no explicit config needed)
```

**Real-World Use Case – E-commerce Site:**

**Tier 1: Web Tier (Public Subnet)**
- **NACL**: Allow inbound 80/443, deny everything else
- **SG**: Allow HTTP/HTTPS from internet, deny SSH except from bastion

**Tier 2: App Tier (Private Subnet)**
- **NACL**: Allow traffic from web tier only (10.0.1.0/24), allow outbound to database
- **SG**: Allow requests from web tier SG (port 8080), allow outbound to database SG

**Tier 3: Database Tier (Isolated Subnet)**
- **NACL**: Allow traffic from app tier only (10.0.2.0/24), deny all inbound from internet
- **SG**: Allow connections from app tier SG only (port 3306)

**Why Both?**
- **NACLs** catch broad, subnet-level attacks (DDoS, subnet scans).
- **Security Groups** provide instance-level granularity.
- **Defense in Depth**: If one is misconfigured, the other still protects.

---

## IAM Scenarios

### Q9. What is the difference between IAM users, groups, roles, and policies?

**Scenario:** You're managing access for a startup with developers, ops engineers, and contractors. How do you organize permissions?

**Answer:**

**IAM User**
- Represents a **person or application** that needs AWS access.
- Has **permanent credentials**: username/password and access keys.
- Cannot be assumed by another user.

Example:
```
User: john.developer
Credentials: Access Key ID + Secret Access Key
Can directly attach policies
```

**IAM Group**
- A **collection of users** for easier permission management.
- Users in a group inherit the group's policies.
- Simplifies management: add user to group instead of attaching policies individually.

Example:
```
Group: Developers
Members: john, jane, mike
Policies: EC2 read/write, S3 access, CloudWatch logs

john is added to group → john gets all developer policies
```

**IAM Role**
- A set of **permissions without a user attached**.
- Assumed **temporarily** by users, applications, or AWS services.
- Has temporary credentials (session token, access key, secret key).
- Useful for cross-account access and EC2 instances.

Example:
```
Role: EC2-S3-Access
Policy: Read/write S3
EC2 instance assumes role → gets temporary credentials → can access S3
```

**IAM Policy**
- A **JSON document defining permissions**.
- Specifies what actions are allowed on which resources.
- Can be attached to users, groups, or roles.

Example policy (allow S3 read-only access):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

**Real-World Organization Example:**

```
Users:
  john.developer (Developer)
  jane.developer (Developer)
  ops-guy.aws (DevOps)
  contractor.external (Contractor)

Groups:
  Developers-Group
    Members: john.developer, jane.developer
    Policy: EC2 full access, Lambda invoke, CloudWatch logs

  Ops-Group
    Members: ops-guy.aws
    Policy: EC2 admin, RDS admin, IAM read-only

  Contractor-Group
    Members: contractor.external
    Policy: S3 read-only for specific bucket

Roles:
  EC2-App-Role
    Policy: S3 read/write, RDS access, CloudWatch
    Used by: EC2 instances (no hardcoded keys)

  Lambda-Role
    Policy: S3 read, DynamoDB write
    Used by: Lambda functions

  Cross-Account-Role
    Policy: Assume from another AWS account
    Used by: Partner company's IAM users
```

**Best Practices:**

1. **Use Groups for Users** – Easier to manage permissions.
2. **Use Roles for EC2/Lambda** – No hardcoded credentials.
3. **Principle of Least Privilege** – Give only needed permissions.
4. **Separate Concerns** – Different groups for different teams.
5. **Use root account sparingly** – Only for account setup.

---

## Bastion Host / Jump Box

### Q10. You have a private subnet in your VPC that contains instances that should not have direct internet access. However, you still need to be able to securely access these instances for administrative purposes. How would you set up a bastion host to facilitate this access?

**Scenario:** Your database servers are in a private subnet with no internet access. You need SSH access for maintenance and debugging.

**Answer:**

Create a **Bastion Host** (Jump Host) in a public subnet as a secure gateway.

**Architecture:**
```
Admin (Your Laptop)
  ↓ SSH
Bastion Host (Public Subnet)
  ↓ SSH (private IP)
Private Instances (Private Subnet)
```

**Step 1: Create Bastion Host EC2 Instance**

- Launch a lightweight EC2 instance (t2.micro) in a **public subnet**.
- Assign a **public IP** or **Elastic IP** (so you can SSH from your laptop).
- Use a minimal OS (Amazon Linux 2, Ubuntu).

**Step 2: Configure Bastion Host Security Group**

```
Bastion SG - Inbound Rules:
  - SSH (22/TCP) from your IP (xxx.xxx.xxx.xxx/32)
    [Restrict to only your IP for security]
  
Bastion SG - Outbound Rules:
  - All traffic (default)
```

**Step 3: Configure Private Instance Security Groups**

```
Private Instance SG - Inbound Rules:
  - SSH (22/TCP) from Bastion SG (sg-bastion)
    [Allow SSH only from bastion, not from internet]
  
Private Instance SG - Outbound Rules:
  - All traffic (default, or restrict as needed)
```

**Step 4: SSH Key Setup**

```
Your Laptop:
  - bastion.pem (private key for bastion)
  - private-instance.pem (private key for private instances)

Bastion Host:
  - Contains private-instance.pem so it can SSH to private instances
  
Private Instance:
  - Authorized public key from private-instance.pem
```

**Step 5: Connect**

From your laptop:
```bash
# SSH into bastion
ssh -i bastion.pem ec2-user@bastion-public-ip

# From bastion, SSH into private instance
ssh -i private-instance.pem ec2-user@10.0.2.50 (private IP)
```

**Or use SSH ProxyCommand (one-liner from your laptop):**
```bash
ssh -i private-instance.pem \
  -o ProxyCommand="ssh -W %h:%p -i bastion.pem ec2-user@bastion-public-ip" \
  ec2-user@10.0.2.50
```

**Better Alternative: AWS Systems Manager Session Manager**

No SSH needed at all:
```bash
# Install Systems Manager agent on EC2 (comes by default)
# Give EC2 instances an IAM role with SSM permissions

aws ssm start-session --target i-1234567890abcdef0
# Now you have shell access without SSH keys
```

**Advantages of Systems Manager:**
- No SSH keys to manage.
- No public IPs needed.
- Audited in CloudTrail.
- Works from AWS Console without command line.

**Security Best Practices for Bastion:**

1. **Restrict SSH access** to your IP/VPN only.
2. **Use key pairs**, not passwords.
3. **Monitor bastion logs** (CloudWatch).
4. **No services running** on bastion (just SSH).
5. **Regular patching** and updates.
6. **Disable password-based auth**.
7. **Use auto-scaling** with a minimal bastion fleet (if many admins).

---

## Quick Interview Tips

**For VPC Questions:**
- Always mention **multiple availability zones** for HA.
- Talk about **security groups** and **NACLs** separately; show you know both.
- Understand **public vs private vs isolated** subnets.
- Be comfortable with **route tables** and **NAT gateways**.

**For IAM Questions:**
- Explain the **principle of least privilege**.
- Show you understand the difference between **users, groups, roles**.
- Mention **cross-account access** scenarios.
- Be aware of **root account** risks.

**For EC2 Questions:**
- Know different **instance types** (t2, m5, c5, r5, p3 and why).
- Understand **security groups** and **key pairs**.
- Discuss **auto-scaling** and **load balancing**.

**General Tips:**
- Draw **architecture diagrams** when explaining.
- Use **real AWS terminology** (don't make up terms).
- Show **hands-on experience** from projects.
- Ask **clarifying questions** if scenario is ambiguous.

Good luck with your AWS interviews!
