# Task 5 — RDS Security Group Dependency

**Status: Done**

## Objective

Build a complete AWS network environment and demonstrate that:

```text
EC2-A → RDS PostgreSQL:5432 → ALLOWED
EC2-B → RDS PostgreSQL:5432 → DENIED
```

The lab is **AWS Management Console only**.

No AWS CLI is required.

You will create:

- VPC
- 2 Availability Zones
- 2 Public Subnets
- 2 Private Subnets
- Internet Gateway
- Public Route Table
- Private Route Table
- EC2-A
- EC2-B
- 3 Security Groups
- RDS DB Subnet Group
- RDS PostgreSQL
- PostgreSQL connectivity tests

---

# 1. Final Architecture

```text
                              INTERNET
                                  |
                           Internet Gateway
                                  |
                    +-------------+-------------+
                    |                           |
              Public Route Table          Private Route Table
                    |                           |
          +---------+---------+         +-------+-------+
          |                   |         |               |
     Public Subnet A     Public Subnet B  Private Subnet A
      10.0.1.0/24         10.0.2.0/24      10.0.11.0/24
          |                   |                   |
        EC2-A               EC2-B                |
     SG-EC2-A             SG-EC2-B                |
          |                   |                   |
          +---------+---------+                   |
                    |                             |
                    | VPC Internal Network        |
                    +-------------+---------------+
                                  |
                           RDS Subnet Group
                                  |
                         +--------+--------+
                         |                 |
                  Private Subnet A   Private Subnet B
                   10.0.11.0/24       10.0.12.0/24
                         |                 |
                         +--------+--------+
                                  |
                            RDS PostgreSQL
                               SG-RDS
                                  |
                              TCP 5432
                                  |
                           Source: SG-EC2-A
```

Expected connectivity:

```text
EC2-A ───────────────► RDS:5432
          ALLOWED ✓

EC2-B ───────────────► RDS:5432
          DENIED ✗
```

---

# 2. Network Design

| Component | Configuration |
|---|---|
| VPC | `10.0.0.0/16` |
| AZ-1 | `ap-south-1a` |
| AZ-2 | `ap-south-1b` |
| Public Subnet A | `10.0.1.0/24` |
| Public Subnet B | `10.0.2.0/24` |
| Private Subnet A | `10.0.11.0/24` |
| Private Subnet B | `10.0.12.0/24` |
| Internet Gateway | Attached to VPC |
| Public Route Table | `0.0.0.0/0 → IGW` |
| Private Route Table | Only VPC local route |
| EC2-A | Public Subnet A |
| EC2-B | Public Subnet B |
| RDS | Private Subnet A + B |
| Database | PostgreSQL |
| PostgreSQL Port | `5432` |
| RDS Source | `SG-EC2-A` only |

---

# 3. Why Four Subnets?

We use:

```text
Public Subnet A
Public Subnet B
Private Subnet A
Private Subnet B
```

because the lab demonstrates a basic multi-AZ architecture.

The two public subnets are used for the EC2 instances:

```text
EC2-A → Public Subnet A
EC2-B → Public Subnet B
```

The two private subnets are used by the RDS DB Subnet Group:

```text
RDS
 |
 +---- Private Subnet A
 |
 +---- Private Subnet B
```

RDS requires a DB subnet group that contains subnets in at least two Availability Zones for the standard Multi-AZ-capable setup.

---

# 4. Prerequisites

You need:

- AWS account
- AWS Management Console access
- Permission to create VPC, EC2, RDS, and Security Groups
- SSH key pair
- Region: **Asia Pacific (Mumbai) — ap-south-1**

---

# 5. Step 1 — Create the VPC

Go to:

```text
AWS Console
→ VPC
→ Your VPCs
→ Create VPC
```

Choose:

```text
Resources to create:
VPC only
```

Name tag:

```text
rds-sg-lab-vpc
```

IPv4 CIDR:

```text
10.0.0.0/16
```

IPv6:

```text
No IPv6 CIDR block
```

Tenancy:

```text
Default
```

Click:

```text
Create VPC
```

---

# 6. Step 2 — Create Public Subnet A

Go to:

```text
VPC
→ Subnets
→ Create subnet
```

Select:

```text
VPC:
rds-sg-lab-vpc
```

Subnet name:

```text
public-subnet-a
```

Availability Zone:

```text
ap-south-1a
```

IPv4 CIDR:

```text
10.0.1.0/24
```

Create the subnet.

---

# 7. Step 3 — Create Public Subnet B

Create another subnet.

```text
VPC:
rds-sg-lab-vpc
```

Name:

```text
public-subnet-b
```

Availability Zone:

```text
ap-south-1b
```

IPv4 CIDR:

```text
10.0.2.0/24
```

Create the subnet.

---

# 8. Step 4 — Create Private Subnet A

Create another subnet.

Name:

```text
private-subnet-a
```

Availability Zone:

```text
ap-south-1a
```

IPv4 CIDR:

```text
10.0.11.0/24
```

Create it.

---

# 9. Step 5 — Create Private Subnet B

Create another subnet.

Name:

```text
private-subnet-b
```

Availability Zone:

```text
ap-south-1b
```

IPv4 CIDR:

```text
10.0.12.0/24
```

Create it.

---

# 10. Final Subnet Layout

You should now have:

```text
VPC: 10.0.0.0/16

AZ: ap-south-1a
    |
    +-- Public A
    |   10.0.1.0/24
    |
    +-- Private A
        10.0.11.0/24


AZ: ap-south-1b
    |
    +-- Public B
    |   10.0.2.0/24
    |
    +-- Private B
        10.0.12.0/24
```

---

# 11. Step 6 — Create Internet Gateway

Go to:

```text
VPC
→ Internet Gateways
→ Create internet gateway
```

Name:

```text
rds-sg-lab-igw
```

Create it.

Then select the Internet Gateway:

```text
Actions
→ Attach to a VPC
```

Select:

```text
rds-sg-lab-vpc
```

Attach it.

---

# 12. Step 7 — Create Public Route Table

Go to:

```text
VPC
→ Route Tables
→ Create route table
```

Name:

```text
public-rt
```

VPC:

```text
rds-sg-lab-vpc
```

Create it.

---

# 13. Step 8 — Add Internet Route

Open:

```text
public-rt
→ Routes
→ Edit routes
→ Add route
```

Add:

```text
Destination:
0.0.0.0/0

Target:
Internet Gateway

Target:
rds-sg-lab-igw
```

Save changes.

The route table should now contain:

```text
Destination        Target
--------------------------------
10.0.0.0/16        local
0.0.0.0/0          IGW
```

---

# 14. Step 9 — Associate Public Subnets

Open:

```text
public-rt
→ Subnet associations
→ Edit subnet associations
```

Select:

```text
public-subnet-a
public-subnet-b
```

Save.

Now:

```text
Public Subnet A
        |
        v
Public Route Table
        |
        v
Internet Gateway
```

and:

```text
Public Subnet B
        |
        v
Public Route Table
        |
        v
Internet Gateway
```

---

# 15. Step 10 — Create Private Route Table

Go to:

```text
VPC
→ Route Tables
→ Create route table
```

Name:

```text
private-rt
```

VPC:

```text
rds-sg-lab-vpc
```

Create it.

For this lab, do not add an Internet Gateway route.

The table should contain only:

```text
Destination        Target
--------------------------------
10.0.0.0/16        local
```

---

# 16. Step 11 — Associate Private Subnets

Open:

```text
private-rt
→ Subnet associations
→ Edit subnet associations
```

Select:

```text
private-subnet-a
private-subnet-b
```

Save.

Now:

```text
Private Subnet A
        |
        v
Private Route Table
        |
        v
No direct Internet route
```

and:

```text
Private Subnet B
        |
        v
Private Route Table
        |
        v
No direct Internet route
```

---

# 17. Do We Need a NAT Gateway?

For this lab:

```text
NO
```

We do not need a NAT Gateway because:

- RDS does not need Internet access for the test
- EC2 instances are in public subnets
- RDS only needs private VPC connectivity

Therefore the architecture avoids unnecessary NAT Gateway cost.

---

# 18. Step 12 — Create EC2-A Security Group

Go to:

```text
EC2
→ Security Groups
→ Create security group
```

Name:

```text
SG-EC2-A
```

Description:

```text
Security group for EC2-A
```

VPC:

```text
rds-sg-lab-vpc
```

Inbound:

```text
SSH
TCP
22
Source: My IP
```

Outbound:

```text
All traffic
0.0.0.0/0
```

Create the security group.

---

# 19. Step 13 — Create EC2-B Security Group

Create another security group.

Name:

```text
SG-EC2-B
```

Description:

```text
Security group for EC2-B
```

VPC:

```text
rds-sg-lab-vpc
```

Inbound:

```text
SSH
TCP
22
Source: My IP
```

Outbound:

```text
All traffic
0.0.0.0/0
```

Create it.

---

# 20. Step 14 — Create RDS Security Group

Create:

```text
SG-RDS
```

VPC:

```text
rds-sg-lab-vpc
```

Inbound rule:

```text
Type:
PostgreSQL

Protocol:
TCP

Port:
5432

Source:
Custom → SG-EC2-A
```

Do NOT add:

```text
0.0.0.0/0
```

Do NOT add:

```text
SG-EC2-B
```

Final RDS inbound rule:

```text
Type        Port       Source
------------------------------------
PostgreSQL  5432       SG-EC2-A
```

Outbound:

```text
All traffic
```

Create the security group.

---

# 21. Step 15 — Launch EC2-A

Go to:

```text
EC2
→ Instances
→ Launch instances
```

Name:

```text
EC2-A
```

AMI:

```text
Ubuntu Server 24.04 LTS
```

Instance type:

```text
t3.micro
```

Select your SSH key pair.

Network settings:

```text
VPC:
rds-sg-lab-vpc

Subnet:
public-subnet-a

Auto-assign Public IP:
Enable
```

Security group:

```text
SG-EC2-A
```

Launch.

---

# 22. Step 16 — Launch EC2-B

Launch another instance.

Name:

```text
EC2-B
```

AMI:

```text
Ubuntu Server 24.04 LTS
```

Instance type:

```text
t3.micro
```

Network:

```text
VPC:
rds-sg-lab-vpc

Subnet:
public-subnet-b

Auto-assign Public IP:
Enable
```

Security group:

```text
SG-EC2-B
```

Launch.

---

# 23. Why Are EC2 Instances Public?

This is for lab convenience.

```text
Internet
   |
   v
IGW
   |
   +---- Public Subnet A → EC2-A
   |
   +---- Public Subnet B → EC2-B
```

This allows you to SSH into both instances easily.

The database remains private:

```text
Private Subnets
      |
      v
RDS PostgreSQL
```

For production, application servers would normally also be private and an ALB/bastion/SSM architecture would be used for administration.

---

# 24. Step 17 — Create RDS DB Subnet Group

Go to:

```text
RDS
→ Subnet groups
→ Create DB subnet group
```

Name:

```text
rds-sg-lab-subnet-group
```

Description:

```text
Private subnet group for RDS Security Group Dependency lab
```

VPC:

```text
rds-sg-lab-vpc
```

Availability Zones:

```text
ap-south-1a
ap-south-1b
```

Subnets:

```text
private-subnet-a
private-subnet-b
```

Create the DB subnet group.

---

# 25. Why RDS Uses Private Subnets

The database should not need a public IP for this lab.

The traffic is:

```text
EC2-A
   |
   | Private VPC traffic
   | TCP 5432
   v
RDS
```

There is no reason for:

```text
Internet
   |
   v
RDS
```

The database remains isolated from direct Internet access.

---

# 26. Step 18 — Create RDS PostgreSQL

Go to:

```text
RDS
→ Databases
→ Create database
```

Choose:

```text
Standard create
```

Engine:

```text
PostgreSQL
```

Choose an available PostgreSQL version.

---

# 27. Database Settings

DB identifier:

```text
rds-postgres-lab
```

Master username:

```text
postgresadmin
```

Create a strong password.

Save the password securely for testing.

---

# 28. RDS Instance Configuration

For a learning lab, select a small/dev configuration available in your account.

For example:

```text
Burstable / small instance
```

Avoid unnecessarily large production-class instances.

---

# 29. RDS Connectivity

Under:

```text
Connectivity
```

Select:

```text
VPC:
rds-sg-lab-vpc
```

DB subnet group:

```text
rds-sg-lab-subnet-group
```

Public access:

```text
No
```

Security group:

```text
Choose existing
SG-RDS
```

Database port:

```text
5432
```

Create the database.

Wait until:

```text
Status:
Available
```

---

# 30. Step 19 — Verify RDS Network Configuration

Open:

```text
RDS
→ Databases
→ rds-postgres-lab
→ Connectivity & security
```

Verify:

```text
VPC:
rds-sg-lab-vpc

Publicly accessible:
No

Port:
5432

Security group:
SG-RDS

DB subnet group:
rds-sg-lab-subnet-group
```

Verify the subnet group contains:

```text
private-subnet-a
private-subnet-b
```

---

# 31. Step 20 — Get the RDS Endpoint

In:

```text
RDS
→ rds-postgres-lab
→ Connectivity & security
```

Find:

```text
Endpoint
Port
```

Example:

```text
Endpoint:
rds-postgres-lab.xxxxxxxxx.ap-south-1.rds.amazonaws.com

Port:
5432
```

Use the actual endpoint shown in your console.

---

# 32. Step 21 — Connect to EC2-A

Connect to EC2-A using:

```text
EC2
→ Instances
→ EC2-A
→ Connect
```

You can use:

```text
SSH
```

or:

```text
EC2 Instance Connect
```

if supported.

---

# 33. Step 22 — Install PostgreSQL Client

On Ubuntu EC2-A, install the PostgreSQL client package.

The package name is:

```text
postgresql-client
```

Repeat the same setup on EC2-B.

No AWS CLI configuration is required for this lab.

---

# 34. Step 23 — Test EC2-A → RDS

From EC2-A, connect to:

```text
RDS Endpoint
Port:
5432
```

Use:

```text
Database:
postgres

Username:
postgresadmin

Password:
Your RDS master password
```

Expected:

```text
Connection successful
```

If you enter the PostgreSQL shell:

```text
postgres=>
```

the connection is working.

---

# 35. Step 24 — Verify EC2-A Database Access

Run:

```sql
SELECT version();
```

Then:

```sql
SELECT current_database();
```

Expected:

```text
postgres
```

This proves:

```text
EC2-A
  |
  | TCP 5432
  v
SG-RDS
  |
  v
RDS PostgreSQL
```

---

# 36. Step 25 — Test EC2-B → RDS

Connect to:

```text
EC2-B
```

Attempt the same PostgreSQL connection.

Use:

```text
RDS Endpoint
Port: 5432
Database: postgres
Username: postgresadmin
Password: same RDS password
```

Expected:

```text
Connection timeout
```

or another network-level connection failure.

The exact error can vary depending on the client and network conditions.

The important result is:

```text
EC2-B → RDS:5432 → DENIED
```

---

# 37. Why EC2-B Is Denied

EC2-A has:

```text
SG-EC2-A
```

RDS allows:

```text
TCP 5432
Source: SG-EC2-A
```

EC2-B has:

```text
SG-EC2-B
```

RDS does not have:

```text
TCP 5432
Source: SG-EC2-B
```

Therefore:

```text
EC2-A → RDS:5432 → ALLOWED
EC2-B → RDS:5432 → DENIED
```

---

# 38. Final Security Group Configuration

## SG-EC2-A

```text
Inbound:
SSH 22
Source: My IP

Outbound:
All traffic
```

## SG-EC2-B

```text
Inbound:
SSH 22
Source: My IP

Outbound:
All traffic
```

## SG-RDS

```text
Inbound:
PostgreSQL
TCP 5432
Source: SG-EC2-A

Outbound:
All traffic
```

---

# 39. Final Route Tables

## Public Route Table

```text
Destination        Target
--------------------------------
10.0.0.0/16        local
0.0.0.0/0          Internet Gateway
```

Associated with:

```text
public-subnet-a
public-subnet-b
```

## Private Route Table

```text
Destination        Target
--------------------------------
10.0.0.0/16        local
```

Associated with:

```text
private-subnet-a
private-subnet-b
```

---

# 40. Complete Traffic Flow

## EC2-A

```text
EC2-A
 |
 | SG-EC2-A
 |
 | TCP 5432
 v
RDS SG
 |
 | Source SG-EC2-A matches
 v
RDS PostgreSQL
```

Result:

```text
ALLOWED ✓
```

## EC2-B

```text
EC2-B
 |
 | SG-EC2-B
 |
 | TCP 5432
 v
RDS SG
 |
 | SG-EC2-B does NOT match
 X
RDS PostgreSQL
```

Result:

```text
DENIED ✗
```

---

# 41. Security Group Reference vs IP Address

Do not configure:

```text
RDS SG
TCP 5432
10.0.1.10/32
```

for this lab.

Instead:

```text
RDS SG
TCP 5432
SG-EC2-A
```

Why?

Suppose EC2-A changes from:

```text
10.0.1.10
```

to:

```text
10.0.1.25
```

The SG-based rule still works because the instance is still associated with:

```text
SG-EC2-A
```

This is a major advantage of Security Group references.

---

# 42. Important Security Group Behavior

Security Groups are:

## Stateful

If:

```text
EC2-A → RDS:5432
```

is allowed, return traffic is automatically allowed.

You do not need to manually create a return rule.

## Allow-Based

Security Groups provide allow rules.

There is no explicit deny rule.

Therefore EC2-B is denied because there is no matching allow rule.

---

# 43. Network Layer vs Database Layer

Two separate checks happen.

```text
                Connection Request
                       |
                       v
              Security Group Check
                       |
              +--------+--------+
              |                 |
           Allowed            Denied
              |
              v
        PostgreSQL Server
              |
              v
       Username/Password
              |
        +-----+-----+
        |           |
      Valid       Invalid
        |           |
        v           v
     Success      Login Fail
```

Therefore:

```text
SG allowed ≠ database authentication automatically succeeds
```

Both must be correct.

---

# 44. Troubleshooting

## EC2-A Cannot Connect

Check:

```text
RDS status = Available
```

Check:

```text
RDS port = 5432
```

Check:

```text
RDS SG = SG-RDS
```

Check SG-RDS inbound:

```text
PostgreSQL
TCP 5432
Source: SG-EC2-A
```

Check:

```text
EC2-A belongs to SG-EC2-A
```

Check:

```text
EC2-A and RDS are in the same VPC
```

Check route tables and NACLs if the Security Group configuration is correct.

---

# 45. EC2-B Can Connect

If EC2-B can connect, inspect all security groups attached to EC2-B and RDS.

Look for:

```text
TCP 5432
0.0.0.0/0
```

or:

```text
TCP 5432
SG-EC2-B
```

or another SG attached to EC2-B that is referenced by SG-RDS.

Remove unnecessary access.

---

# 46. EC2-A Reaches the Database but Login Fails

If the network connection reaches PostgreSQL but authentication fails, check:

```text
Username
Password
Database name
PostgreSQL authentication
```

This is different from Security Group access.

---

# 47. Why No NAT Gateway?

This lab intentionally does not use NAT Gateway.

Reason:

```text
EC2:
Public subnets
```

so they can be accessed for testing.

```text
RDS:
Private subnets
```

and RDS does not need Internet access for the database connection.

Therefore:

```text
NAT Gateway = Not required
```

This also avoids unnecessary NAT Gateway charges during the lab.

---

# 48. Production Architecture Improvement

For production, you would normally avoid putting application EC2 instances directly in public subnets.

A more typical design is:

```text
                     INTERNET
                         |
                        ALB
                         |
                +--------+--------+
                |                 |
          Private EC2-A     Private EC2-B
                |                 |
                +--------+--------+
                         |
                      TCP 5432
                         |
                   Private RDS
```

Security Groups:

```text
ALB-SG
   |
   v
APP-SG
   |
   v
RDS-SG
```

RDS:

```text
RDS-SG
Inbound:
5432
Source: APP-SG
```

This creates a clean three-tier Security Group dependency.

---

# 49. Final Test Matrix

| Test | Expected Result |
|---|---|
| EC2-A → RDS:5432 | **ALLOWED** |
| EC2-B → RDS:5432 | **DENIED** |
| EC2-A → PostgreSQL login | **SUCCESS** with valid credentials |
| EC2-B → PostgreSQL login | **BLOCKED at network layer** |
| Internet → RDS:5432 | **DENIED** |
| RDS public access | **No** |

---

# 50. Cleanup

## Delete RDS

Go to:

```text
RDS
→ Databases
→ rds-postgres-lab
→ Actions
→ Delete
```

For a temporary lab, select the appropriate option to avoid keeping an unnecessary final snapshot.

Be careful: RDS deletion is destructive.

---

## Delete DB Subnet Group

After the RDS database is deleted:

```text
RDS
→ Subnet groups
→ rds-sg-lab-subnet-group
→ Delete
```

---

## Terminate EC2-A

```text
EC2
→ Instances
→ EC2-A
→ Instance state
→ Terminate
```

---

## Terminate EC2-B

```text
EC2
→ Instances
→ EC2-B
→ Instance state
→ Terminate
```

---

## Delete Security Groups

After dependent resources are removed:

```text
EC2
→ Security Groups
```

Delete:

```text
SG-EC2-A
SG-EC2-B
SG-RDS
```

---

## Delete Route Tables

Delete:

```text
public-rt
private-rt
```

Only after their subnet associations are removed.

---

## Detach and Delete Internet Gateway

Go to:

```text
VPC
→ Internet Gateways
→ rds-sg-lab-igw
→ Actions
→ Detach from VPC
```

Then delete the Internet Gateway.

---

## Delete Subnets

Delete:

```text
public-subnet-a
public-subnet-b
private-subnet-a
private-subnet-b
```

---

## Delete VPC

Finally:

```text
VPC
→ Your VPCs
→ rds-sg-lab-vpc
→ Actions
→ Delete VPC
```

---

# 51. Interview Questions

### Q1. Why is RDS placed in private subnets?

To prevent direct Internet access and keep database connectivity inside the VPC.

### Q2. Why do we use two private subnets?

RDS DB subnet groups are designed across multiple Availability Zones for resilient database deployment.

### Q3. Why use a Security Group reference instead of an IP?

The access relationship follows the Security Group association instead of one specific IP address.

### Q4. Can Security Groups have explicit deny rules?

No. Security Groups use allow rules. Traffic without a matching allow is not permitted.

### Q5. Are Security Groups stateful?

Yes.

### Q6. What port does PostgreSQL use?

TCP:

```text
5432
```

### Q7. Why does EC2-A work but EC2-B fail?

Because:

```text
SG-RDS
TCP 5432
Source = SG-EC2-A
```

and EC2-B belongs to:

```text
SG-EC2-B
```

which is not allowed.

### Q8. Does the RDS Security Group need an Internet Gateway route?

No.

RDS communicates with EC2 using private VPC networking.

### Q9. Does this lab need a NAT Gateway?

No.

The database does not require outbound Internet connectivity for this test.

### Q10. What is the complete traffic path?

For EC2-A:

```text
EC2-A
→ SG-EC2-A
→ VPC local route
→ RDS ENI
→ SG-RDS
→ TCP 5432
→ PostgreSQL
```

For EC2-B:

```text
EC2-B
→ SG-EC2-B
→ VPC local route
→ RDS ENI
→ SG-RDS
→ No matching 5432 allow
→ DENIED
```

---

# 52. Final Architecture Summary

```text
                              INTERNET
                                  |
                           Internet Gateway
                                  |
                    +-------------+-------------+
                    |                           |
              Public Route Table          Private Route Table
                    |                           |
          +---------+---------+         +-------+-------+
          |                   |         |               |
      Public-A            Public-B   Private-A      Private-B
    10.0.1.0/24         10.0.2.0/24 10.0.11.0/24  10.0.12.0/24
          |                   |         |               |
        EC2-A               EC2-B      +-------+-------+
     SG-EC2-A             SG-EC2-B             |
          |                   |          RDS Subnet Group
          |                   |                 |
          +---------+---------+                 |
                    |                           |
                    | VPC local routing        |
                    +-------------+-------------+
                                  |
                           RDS PostgreSQL
                              SG-RDS
                                  |
                              TCP 5432
                                  |
                           Source: SG-EC2-A
```

## Final Verification

```text
EC2-A
  |
  | TCP 5432
  v
RDS PostgreSQL
  ✓ ALLOWED


EC2-B
  |
  | TCP 5432
  v
RDS PostgreSQL
  ✗ DENIED
```

## Key Lesson

> **The network path makes the connection possible, but the RDS Security Group determines which source is allowed to reach PostgreSQL.**

The complete dependency is:

```text
VPC
 ↓
Subnets
 ↓
Route Tables
 ↓
EC2
 ↓
Security Group
 ↓
RDS Subnet Group
 ↓
RDS PostgreSQL
 ↓
RDS Security Group
 ↓
TCP 5432
 ↓
SG-EC2-A only
```
