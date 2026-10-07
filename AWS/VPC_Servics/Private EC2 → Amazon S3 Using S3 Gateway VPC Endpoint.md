# Private EC2 → Amazon S3 Using S3 Gateway VPC Endpoint

## 1. Lab Objective

In this lab, we will configure a **private EC2 instance** to communicate with an **Amazon S3 bucket** using an **S3 Gateway VPC Endpoint**.

The final EC2 instance will:

- Have only a private IP address
- Have no public IPv4 address
- Have no Internet Gateway route
- Have no NAT Gateway
- Use an IAM Role instead of hard-coded AWS credentials
- Access S3 through an S3 Gateway VPC Endpoint
- Upload files to S3
- Download files from S3
- Delete objects from S3

### Final architecture

<img width="1312" height="1199" alt="image" src="https://github.com/user-attachments/assets/c181a355-cfa6-4f59-8efe-e384346a9592" />


---

# 2. What We Are Going to Create

| Component | Value |
|---|---|
| AWS Region | `ap-south-1` |
| VPC | `private-ec2-s3-vpc` |
| VPC CIDR | `10.0.0.0/16` |
| Private Subnet | `private-subnet` |
| Subnet CIDR | `10.0.1.0/24` |
| Route Table | `private-route-table` |
| EC2 | Ubuntu |
| EC2 Public IP | None |
| EC2 Private IP | `10.0.1.x` |
| Security Group | `private-ec2-sg` |
| IAM Role | `PrivateEC2S3Role` |
| VPC Endpoint | `s3-gateway-endpoint` |
| Endpoint Type | Gateway |
| S3 Bucket | `private-ec2-s3-demo-<unique>` |

> **Important:** S3 bucket names are globally unique. Replace the example bucket name with your own unique name.

---

# 3. Prerequisites

You need:

- AWS account
- Permission to create:
  - VPC resources
  - EC2 instances
  - IAM roles/policies
  - S3 buckets
  - VPC endpoints
- AWS Region selected as `ap-south-1`

Optional:

- AWS CLI on your local computer
- Systems Manager Session Manager access to the EC2 instance

---

# 4. Important Concept

There are two separate things happening:

```text
NETWORK CONNECTIVITY
--------------------

EC2
 ↓
Route Table
 ↓
S3 Gateway Endpoint
 ↓
S3
```

and:

```text
AUTHORIZATION
-------------

EC2
 ↓
IAM Role
 ↓
S3 permissions
```

Both are required.

A perfect network path does not automatically give the EC2 permission to access the bucket.

---

# 5. Why Use an S3 Gateway Endpoint?

Without an endpoint, a private EC2 commonly needs:

```text
Private EC2
     |
     v
NAT Gateway
     |
     v
Internet Gateway
     |
     v
S3
```

For S3 access, this is often unnecessary.

With an S3 Gateway Endpoint:

```text
Private EC2
     |
     v
Route Table
     |
     v
S3 Gateway Endpoint
     |
     v
S3
```

For this lab:

```text
NAT Gateway = NOT REQUIRED
Internet Gateway = NOT REQUIRED
Public IP = NOT REQUIRED
```

---

# 6. Gateway Endpoint vs Interface Endpoint

This distinction is important.

## Gateway Endpoint

Used for:

```text
S3
DynamoDB
```

The route table is used to direct traffic.

```text
EC2
 ↓
Route Table
 ↓
Prefix List
 ↓
Gateway Endpoint
 ↓
AWS Service
```

## Interface Endpoint

Uses an ENI/private IP address in your subnet.

```text
EC2
 ↓
Private IP
 ↓
Endpoint ENI
 ↓
AWS Service
```

For this lab:

```text
Private EC2 → S3
```

we use:

```text
S3 Gateway Endpoint
```

---

# 7. Step 1 — Select the AWS Region

Open the AWS Console.

Select:

```text
Asia Pacific (Mumbai)
ap-south-1
```

Keep the following resources in this region for this lab:

```text
VPC
EC2
S3 Gateway Endpoint
```

For simplicity, create the S3 bucket in the same region as well.

---

# 8. Step 2 — Create the VPC

Go to:

```text
AWS Console
    ↓
VPC
    ↓
Your VPCs
    ↓
Create VPC
```

Select:

```text
Resources to create:
VPC only
```

Enter:

```text
Name:
private-ec2-s3-vpc

IPv4 CIDR:
10.0.0.0/16
```

Create the VPC.

---

# 9. Step 3 — Verify the VPC

Open:

```text
VPC
 ↓
Your VPCs
```

Verify:

```text
Name:
private-ec2-s3-vpc

IPv4 CIDR:
10.0.0.0/16
```

---

# 10. Step 4 — Create the Private Subnet

Go to:

```text
VPC
 ↓
Subnets
 ↓
Create subnet
```

Select:

```text
VPC:
private-ec2-s3-vpc
```

Enter:

```text
Subnet name:
private-subnet

Availability Zone:
ap-south-1a

IPv4 subnet CIDR:
10.0.1.0/24
```

Create the subnet.

---

# 11. What Makes This a Private Subnet?

Do not assume that naming a subnet `private-subnet` makes it private.

The route table determines whether it has a route to the internet.

A private route table will not contain:

```text
0.0.0.0/0 → Internet Gateway
```

We will create the correct route table next.

---

# 12. Step 5 — Create a Route Table

Go to:

```text
VPC
 ↓
Route tables
 ↓
Create route table
```

Enter:

```text
Name:
private-route-table

VPC:
private-ec2-s3-vpc
```

Create the route table.

---

# 13. Step 6 — Associate the Route Table With the Private Subnet

Open:

```text
private-route-table
```

Go to:

```text
Subnet associations
 ↓
Edit subnet associations
```

Select:

```text
private-subnet
```

Save.

The relationship is now:

```text
private-subnet
       |
       v
private-route-table
```

---

# 14. Step 7 — Verify the Route Table

Open:

```text
private-route-table
 ↓
Routes
```

Initially you should see:

```text
Destination       Target
--------------------------------
10.0.0.0/16       local
```

You should NOT see:

```text
0.0.0.0/0 → Internet Gateway
```

This means there is no direct internet route.

---

# 15. Step 8 — Create the S3 Bucket

Go to:

```text
AWS Console
 ↓
S3
 ↓
Create bucket
```

Example:

```text
Bucket name:

private-ec2-s3-demo-20261007-12345
```

Use your own unique name.

Region:

```text
Asia Pacific (Mumbai)
ap-south-1
```

Keep:

```text
Block all public access:
Enabled
```

Create the bucket.

---

# 16. Step 9 — Verify the S3 Bucket

Open:

```text
S3
 ↓
Buckets
```

Verify:

```text
private-ec2-s3-demo-20261007-12345
```

The bucket should remain private.

We are NOT making the bucket public.

---

# 17. Step 10 — Create the IAM Role for EC2

The EC2 instance needs permission to access S3.

Go to:

```text
IAM
 ↓
Roles
 ↓
Create role
```

Select:

```text
Trusted entity:
AWS service
```

Select:

```text
Use case:
EC2
```

Continue.

---

# 18. Step 11 — Create the S3 IAM Policy

For this lab, use a least-privilege custom policy.

Create a policy containing:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListSpecificBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME"
    },
    {
      "Sid": "ObjectAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
    }
  ]
}
```

Replace:

```text
YOUR-BUCKET-NAME
```

with your actual bucket name.

For example:

```text
private-ec2-s3-demo-20261007-12345
```

---

# 19. Why Are There Two S3 ARNs?

This is important.

The bucket itself:

```text
arn:aws:s3:::my-bucket
```

Objects inside the bucket:

```text
arn:aws:s3:::my-bucket/*
```

`ListBucket` applies to the bucket.

Object operations apply to objects:

```text
GetObject
PutObject
DeleteObject
```

Therefore we need both resources.

---

# 20. Step 12 — Name the IAM Role

Use:

```text
PrivateEC2S3Role
```

Attach the custom S3 policy.

Create the role.

---

# 21. Step 13 — Create the EC2 Security Group

Go to:

```text
EC2
 ↓
Security Groups
 ↓
Create security group
```

Enter:

```text
Security group name:
private-ec2-sg

Description:
Security group for private EC2 S3 lab

VPC:
private-ec2-s3-vpc
```

---

# 22. Security Group Inbound Rules

For S3 access, you do not need an inbound security-group rule.

You can leave inbound empty for this lab:

```text
Inbound:
No rules
```

If you need SSH or another application connection, add only the required source/rule.

---

# 23. Security Group Outbound Rules

For the lab, leave the default outbound rule:

```text
Type:
All traffic

Destination:
0.0.0.0/0
```

This does NOT mean the EC2 has internet access.

A security group's outbound rule allows traffic, but the subnet still needs an actual route to the destination.

Our private subnet does not have an Internet Gateway route.

---

# 24. Step 14 — Launch the EC2 Instance

Go to:

```text
EC2
 ↓
Instances
 ↓
Launch instance
```

Name:

```text
private-s3-test-ec2
```

AMI:

```text
Ubuntu Server
```

Instance type:

```text
t3.micro
```

Use another instance type if required.

---

# 25. Step 15 — Configure EC2 Networking

Under Network settings:

```text
VPC:
private-ec2-s3-vpc
```

Subnet:

```text
private-subnet
```

Auto-assign Public IP:

```text
Disable
```

Security Group:

```text
private-ec2-sg
```

---

# 26. Step 16 — Attach the IAM Role

Under the EC2 advanced details:

```text
IAM instance profile
```

Select:

```text
PrivateEC2S3Role
```

Launch the instance.

---

# 27. Step 17 — Verify the EC2

Open:

```text
EC2
 ↓
Instances
 ↓
private-s3-test-ec2
```

Verify:

```text
Private IPv4:
10.0.1.x

Public IPv4:
None

Subnet:
private-subnet

VPC:
private-ec2-s3-vpc
```

The EC2 should have no public IP.

---

# 28. Step 18 — Understand the EC2 Network Path Before Creating the Endpoint

At this point:

```text
Private EC2
     |
     v
Private Route Table
     |
     v
Only:
10.0.0.0/16 → local
```

There is no route to S3.

Therefore:

```text
EC2 → S3
```

does not yet have the required network path.

We will create that path using the S3 Gateway Endpoint.

---

# 29. Step 19 — Create the S3 Gateway VPC Endpoint

Go to:

```text
VPC
 ↓
Endpoints
 ↓
Create endpoint
```

Name:

```text
s3-gateway-endpoint
```

---

# 30. Step 20 — Select AWS Services

For Service category:

```text
AWS services
```

Search for:

```text
S3
```

Select the S3 service for your region.

It will look similar to:

```text
com.amazonaws.ap-south-1.s3
```

Make sure the endpoint type is:

```text
Gateway
```

---

# 31. Step 21 — Select the VPC

Select:

```text
private-ec2-s3-vpc
```

---

# 32. Step 22 — Select the Route Table

Under Route Tables, select:

```text
private-route-table
```

This is one of the most important steps.

The endpoint must be associated with the route table used by the private subnet.

The relationship should be:

```text
Private Subnet
      |
      v
Private Route Table
      |
      v
S3 Gateway Endpoint
```

Create the endpoint.

---

# 33. Step 23 — Verify the Endpoint

Go to:

```text
VPC
 ↓
Endpoints
```

Find:

```text
s3-gateway-endpoint
```

Verify:

```text
Type:
Gateway

Service:
com.amazonaws.ap-south-1.s3

State:
Available
```

---

# 34. Step 24 — Check the Route Table Again

Open:

```text
VPC
 ↓
Route tables
 ↓
private-route-table
 ↓
Routes
```

You should now see a route similar to:

```text
Destination                 Target
------------------------------------------------
10.0.0.0/16                 local
S3 prefix list (pl-xxxx)    vpce-xxxx
```

The exact prefix-list ID and endpoint ID will be different in your account.

---

# 35. The Most Important Networking Concept

The S3 Gateway Endpoint does not behave like an EC2 network interface with a private IP.

You do NOT normally have:

```text
EC2
 ↓
10.0.2.50
 ↓
S3 Endpoint
```

Instead:

```text
EC2
 ↓
Route Table
 ↓
S3 Prefix List
 ↓
Gateway Endpoint
 ↓
S3
```

The route table is what tells the VPC:

> Traffic destined for S3 should use this gateway endpoint.

---

# 36. Step 25 — Configure Endpoint Policy

The endpoint has a policy that can control what S3 requests are allowed through it.

For the first lab, you can keep the endpoint policy broad and rely on the IAM role for authorization.

Conceptually:

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": "*"
    }
  ]
}
```

This does NOT make your S3 bucket public.

The endpoint policy controls access through the endpoint.

IAM still determines whether the EC2 role can perform the requested action.

---

# 37. Optional — Restrictive Endpoint Policy

After the basic lab works, you can restrict the endpoint to your bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSpecificBucket",
      "Effect": "Allow",
      "Principal": "*",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME"
    },
    {
      "Sid": "AllowSpecificBucketObjects",
      "Effect": "Allow",
      "Principal": "*",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
    }
  ]
}
```

Replace:

```text
YOUR-BUCKET-NAME
```

with your bucket name.

---

# 38. IAM Policy vs Endpoint Policy vs Bucket Policy

These are different controls.

## IAM Policy

Attached to the EC2 IAM role.

It answers:

```text
Is this IAM principal allowed to perform this S3 operation?
```

## VPC Endpoint Policy

Attached to the endpoint.

It answers:

```text
Is this request allowed through this endpoint?
```

## S3 Bucket Policy

Attached to the bucket.

It answers:

```text
Does the bucket allow this request?
```

Think:

```text
EC2
 |
 | IAM Role
 v
IAM Policy
 |
 v
S3 Request
 |
 +----> Endpoint Policy
 |
 +----> Bucket Policy
 |
 v
S3
```

An explicit `Deny` can override an `Allow`.

---

# 39. Step 26 — Access the Private EC2

Because the EC2 has no public IP, you cannot directly SSH to it from the internet.

Recommended option:

```text
AWS Systems Manager Session Manager
```

Go to:

```text
EC2
 ↓
Instances
 ↓
private-s3-test-ec2
 ↓
Connect
 ↓
Session Manager
```

Click:

```text
Connect
```

---

# 40. Important SSM Note

The S3 endpoint does NOT automatically provide SSM connectivity.

These are different services:

```text
S3
 ↓
S3 Gateway Endpoint
```

and:

```text
SSM
 ↓
SSM connectivity
```

If your VPC has no NAT Gateway and you want to use Session Manager, you need an appropriate private connectivity design for SSM, commonly involving SSM-related Interface VPC Endpoints and the required IAM permissions.

Do not troubleshoot SSM as though it were an S3 problem.

---

# 41. Step 27 — Install AWS CLI on EC2

Inside the EC2:

```bash
aws --version
```

If AWS CLI is not installed:

```bash
sudo apt update
```

Then install it using the AWS CLI installation method appropriate for your Ubuntu image.

If the package is available:

```bash
sudo apt install -y awscli
```

Verify:

```bash
aws --version
```

---

# 42. Step 28 — Verify the EC2 IAM Role

Run:

```bash
aws sts get-caller-identity
```

Expected result should identify the EC2 IAM role.

Example:

```json
{
    "UserId": "AROAxxxxxxxxxxxx",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:role/PrivateEC2S3Role"
}
```

This confirms that the AWS CLI is using the EC2 IAM role.

---

# 43. Do NOT Configure Access Keys

Do NOT do this on the EC2:

```bash
aws configure
```

with:

```text
AWS Access Key ID
AWS Secret Access Key
```

for this lab.

The preferred design is:

```text
EC2
 ↓
IAM Instance Role
 ↓
Temporary credentials
 ↓
AWS CLI
 ↓
S3
```

---

# 44. Step 29 — Set the AWS Region

Check:

```bash
aws configure get region
```

If required:

```bash
export AWS_DEFAULT_REGION=ap-south-1
```

Verify:

```bash
echo $AWS_DEFAULT_REGION
```

Expected:

```text
ap-south-1
```

---

# 45. Step 30 — Test S3 Access

Use the specific bucket:

```bash
aws s3 ls s3://YOUR-BUCKET-NAME
```

Example:

```bash
aws s3 ls s3://private-ec2-s3-demo-20261007-12345
```

If the IAM permissions, route table and endpoint are correct, this should succeed.

---

# 46. Step 31 — Create a Test File

On the EC2:

```bash
echo "Hello from private EC2" > test.txt
```

Verify:

```bash
cat test.txt
```

Expected:

```text
Hello from private EC2
```

---

# 47. Step 32 — Upload the File to S3

Run:

```bash
aws s3 cp test.txt s3://YOUR-BUCKET-NAME/
```

Example:

```bash
aws s3 cp test.txt \
s3://private-ec2-s3-demo-20261007-12345/
```

Expected:

```text
upload: ./test.txt to s3://...
```

---

# 48. Step 33 — Verify the Object

Run:

```bash
aws s3 ls s3://YOUR-BUCKET-NAME/
```

You should see:

```text
test.txt
```

The data path was:

```text
EC2
 ↓
Private Route Table
 ↓
S3 Prefix List
 ↓
S3 Gateway Endpoint
 ↓
S3
```

---

# 49. Step 34 — Download the Object

Delete the local file:

```bash
rm test.txt
```

Verify:

```bash
ls
```

Now download:

```bash
aws s3 cp \
s3://YOUR-BUCKET-NAME/test.txt \
.
```

Verify:

```bash
cat test.txt
```

Expected:

```text
Hello from private EC2
```

---

# 50. Step 35 — Delete the S3 Object

Run:

```bash
aws s3 rm s3://YOUR-BUCKET-NAME/test.txt
```

Verify:

```bash
aws s3 ls s3://YOUR-BUCKET-NAME/
```

The object should no longer appear.

---

# 51. Step 36 — Verify That EC2 Has No Public IP

From the AWS Console:

```text
EC2
 ↓
Instances
 ↓
private-s3-test-ec2
```

Check:

```text
Public IPv4 address:
None
```

Check:

```text
Private IPv4 address:
10.0.1.x
```

---

# 52. Step 37 — Verify the Private Route

Open:

```text
VPC
 ↓
Route Tables
 ↓
private-route-table
 ↓
Routes
```

You should have something similar to:

```text
Destination                 Target
------------------------------------------------
10.0.0.0/16                 local
pl-xxxxxxxx                 vpce-xxxxxxxx
```

The second route is the important S3 route.

---

# 53. Step 38 — Understand the Complete Request Flow

Suppose you run:

```bash
aws s3 cp test.txt s3://my-bucket/
```

The flow is:

```text
1. AWS CLI
       |
       v
2. EC2 IAM Role
       |
       | Temporary credentials
       v
3. S3 API request
       |
       v
4. VPC networking
       |
       v
5. Private Route Table
       |
       | Destination matches S3 prefix list
       v
6. S3 Gateway Endpoint
       |
       v
7. AWS network
       |
       v
8. Amazon S3
       |
       v
9. IAM / bucket / endpoint authorization
       |
       v
10. Object operation
```

---

# 54. Very Important: Network vs Permission

Suppose this works:

```bash
aws sts get-caller-identity
```

but this fails:

```bash
aws s3 ls s3://my-bucket
```

Your network may be working perfectly.

The problem could be:

```text
IAM permissions
Endpoint policy
Bucket policy
```

Conversely, if the IAM policy is correct but there is no S3 route:

```text
IAM = Allow
Network = Broken
```

the request still fails.

Therefore troubleshoot these separately.

---

# 55. Network Layer

Check:

```text
EC2
 ↓
Subnet
 ↓
Route Table
 ↓
S3 Prefix List
 ↓
Gateway Endpoint
 ↓
S3
```

---

# 56. Authorization Layer

Check:

```text
EC2 IAM Role
 ↓
IAM Policy
 ↓
Endpoint Policy
 ↓
Bucket Policy
 ↓
S3
```

---

# 57. Why Doesn't the EC2 Need an Internet Gateway?

Because S3 Gateway Endpoint provides a private VPC path.

The EC2 does not need:

```text
EC2
 ↓
Internet
 ↓
S3
```

It uses:

```text
EC2
 ↓
VPC Route Table
 ↓
S3 Gateway Endpoint
 ↓
S3
```

---

# 58. Why Doesn't the EC2 Need a NAT Gateway?

For this S3 communication, no NAT Gateway is required.

NAT Gateway is generally used when private resources need outbound connectivity to destinations outside the VPC through public endpoints.

For S3:

```text
Private EC2
      |
      v
S3 Gateway Endpoint
      |
      v
S3
```

is sufficient.

---

# 59. What If the EC2 Needs the Internet Too?

Suppose your application needs:

```text
S3
GitHub
Docker Hub
Ubuntu repositories
External APIs
Public websites
```

Then an S3 endpoint alone isn't enough.

You could use:

```text
                     Private EC2
                         |
                 +-------+-------+
                 |               |
                 v               v
          S3 Gateway         NAT Gateway
           Endpoint               |
                 |                v
                 v           Internet
                 S3
```

This is a common production design.

---

# 60. S3 Gateway Endpoint Route

The most important route looks conceptually like:

```text
Destination:
S3 prefix list

Target:
vpce-xxxxxxxx
```

The prefix list represents the AWS-managed network destinations for S3.

You do not manually enter S3 IP addresses.

---

# 61. Do NOT Hard-Code S3 IP Addresses

Do not create routes such as:

```text
52.x.x.x/32
```

for S3.

S3 uses a large, distributed infrastructure.

AWS provides the managed prefix list and endpoint mechanism for this purpose.

---

# 62. Troubleshooting

## Problem 1 — `AccessDenied`

Example:

```text
An error occurred (AccessDenied)
```

Check IAM permissions.

For bucket listing:

```text
s3:ListBucket
```

Resource:

```text
arn:aws:s3:::YOUR-BUCKET-NAME
```

For objects:

```text
s3:GetObject
s3:PutObject
s3:DeleteObject
```

Resource:

```text
arn:aws:s3:::YOUR-BUCKET-NAME/*
```

---

# 63. Problem 2 — Endpoint Exists but Request Fails

Check:

```text
VPC
 ↓
Endpoints
 ↓
s3-gateway-endpoint
```

State must be:

```text
Available
```

Then check the endpoint's route-table association.

---

# 64. Problem 3 — Wrong Route Table

This is a very common mistake.

Incorrect:

```text
Private Subnet
     |
     v
Route Table A

S3 Endpoint
     |
     v
Route Table B
```

Correct:

```text
Private Subnet
     |
     v
Private Route Table
     |
     v
S3 Gateway Endpoint
```

The endpoint must be associated with the route table used by the subnet.

---

# 65. Problem 4 — Wrong VPC

Make sure:

```text
EC2 VPC
=
Endpoint VPC
```

For this lab:

```text
private-ec2-s3-vpc
```

must be the VPC selected when creating the endpoint.

---

# 66. Problem 5 — Wrong Region

For this lab:

```text
VPC:
ap-south-1

EC2:
ap-south-1

S3 Endpoint:
ap-south-1
```

Do not accidentally create the VPC endpoint in another region.

---

# 67. Problem 6 — Security Group

Check the EC2 security group's outbound rules.

For the lab:

```text
Outbound:
All traffic
0.0.0.0/0
```

is acceptable.

Remember:

```text
Security Group = Stateful
```

---

# 68. Problem 7 — Custom Network ACL

If you are using a custom NACL, verify that it allows the required outbound and return traffic.

Remember:

```text
NACL = Stateless
```

Unlike security groups, return traffic must be explicitly permitted by the appropriate NACL rules.

---

# 69. Problem 8 — Bucket Policy Denies Access

Check:

```text
S3
 ↓
Bucket
 ↓
Permissions
 ↓
Bucket policy
```

Look for explicit:

```text
"Deny"
```

An explicit deny can override an allow.

---

# 70. Problem 9 — Endpoint Policy Denies Access

Check:

```text
VPC
 ↓
Endpoints
 ↓
S3 Endpoint
 ↓
Policy
```

Make sure the endpoint policy permits the required action and bucket.

---

# 71. Problem 10 — SSM Connection Fails

If:

```text
Session Manager
```

doesn't connect, don't assume the S3 endpoint is broken.

SSM and S3 have different connectivity requirements.

```text
S3
 ↓
S3 Gateway Endpoint

SSM
 ↓
SSM connectivity
```

For a completely private EC2 without NAT, configure the required SSM connectivity separately.

---

# 72. Optional Experiment — Remove the S3 Endpoint

This is useful to understand the architecture.

First verify S3 works:

```bash
aws s3 ls s3://YOUR-BUCKET-NAME
```

Then temporarily remove the S3 endpoint or its route.

Your route table will return to:

```text
10.0.0.0/16 → local
```

Now the private EC2 has no route to S3.

Try:

```bash
aws s3 ls s3://YOUR-BUCKET-NAME
```

The request should fail or time out because there is no suitable network path.

Restore the endpoint afterward.

---

# 73. Optional Experiment — Add NAT Gateway

Now imagine the architecture:

```text
Private EC2
     |
     v
NAT Gateway
     |
     v
Internet Gateway
     |
     v
Internet
```

The EC2 can now reach public destinations.

But for S3, the S3 Gateway Endpoint gives a more direct AWS VPC path:

```text
Private EC2
     |
     v
S3 Gateway Endpoint
     |
     v
S3
```

A production VPC may use both.

---

# 74. Optional — Restrict the S3 Bucket to the VPC Endpoint

For additional control, you can use an S3 bucket policy with the `aws:sourceVpce` condition.

Example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRequestsNotFromExpectedVPCEndpoint",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::YOUR-BUCKET-NAME",
        "arn:aws:s3:::YOUR-BUCKET-NAME/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpce": "vpce-xxxxxxxxxxxxxxxxx"
        }
      }
    }
  ]
}
```

Replace:

```text
YOUR-BUCKET-NAME
```

and:

```text
vpce-xxxxxxxxxxxxxxxxx
```

with your actual values.

### Warning

Do not blindly apply this to an important production bucket.

A bucket policy like this can block legitimate access that does not come through that endpoint.

Test it carefully.

---

# 75. Production Security Recommendations

For production:

## EC2

Use:

```text
IAM Role
```

instead of access keys.

## S3

Enable:

```text
Block Public Access
```

and use appropriate:

```text
Encryption
Versioning
Lifecycle policies
Logging/monitoring
```

## IAM

Use least privilege.

Don't automatically give:

```text
s3:*
```

to every application.

## Endpoint

Use an appropriate endpoint policy.

## Bucket

Use a bucket policy when additional restrictions are required.

---

# 76. Cost Consideration

One important advantage of the S3 Gateway Endpoint is that it avoids sending this S3 traffic through a NAT Gateway.

Typical design:

```text
Private EC2
     |
     +--------> S3 Gateway Endpoint --------> S3
     |
     +--------> NAT Gateway -----------------> Internet
```

This allows you to reserve NAT Gateway usage for traffic that actually needs general internet egress.

Always verify current AWS pricing before designing a production architecture.

---

