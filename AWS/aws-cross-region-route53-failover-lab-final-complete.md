# AWS Cross-Region DNS Failover -- End-to-End High Availability Lab



---

# 0.1 Account/Region Checklist

Before creating anything, keep two AWS Console sessions available.

### Account A

```text
Purpose: Primary
Region: ap-south-1
Region name: Asia Pacific (Mumbai)
```

### Account B

```text
Purpose: Secondary/DR
Region: ap-south-2
Region name: Asia Pacific (Hyderabad)
```

Confirm the active AWS account and region before every regional operation.

Optional CLI verification:

```bash
aws sts get-caller-identity
aws configure get region
```

Do not create the Mumbai resources while the Console is still switched to Hyderabad, or vice versa.

---

## 1. Project Overview

This hands-on project implements an **active-passive high-availability
application architecture across two AWS accounts and two AWS regions**.

The application is hosted on EC2 instances running Nginx. Each region
has an Application Load Balancer (ALB). Amazon Route 53 provides
DNS-based failover between the primary and secondary environments.

### Primary environment

-   AWS Account A
-   Region: Mumbai -- `ap-south-1`
-   VPC: `mum-prod-vpc`
-   VPC CIDR: `10.10.0.0/16`
-   Application: Nginx on EC2
-   ALB: `mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com`

### Secondary environment

-   AWS Account B
-   Region: Hyderabad -- `ap-south-2`
-   VPC: `hyd-dr`
-   VPC CIDR: `10.20.0.0/16`
-   Application: Nginx on EC2
-   ALB: `hyd-alb-1968825055.ap-south-2.elb.amazonaws.com`

### DNS

-   Domain: `demodomain.fun`
-   Application hostname: `app.demodomain.fun`
-   DNS provider: Amazon Route 53
-   Domain registrar: Hostinger
-   Routing policy: Failover
-   Primary: Mumbai
-   Secondary: Hyderabad
-   TTL: `60` seconds

### Important scope decisions

This lab intentionally does **not** include:

-   RDS/database deployment
-   Database replication
-   AWS KMS database encryption setup
-   WAF
-   ACM/HTTPS
-   A third DR region
-   Cross-region application data replication

The focus is:

> VPC → EC2 → Nginx → ALB → Route 53 Health Check → Route 53 Failover →
> DNS-based application failover.

------------------------------------------------------------------------

# 2. Final Architecture

``` text
                         Internet
                            |
                            |
                  app.demodomain.fun
                            |
                            v
                    Amazon Route 53
                    Failover Routing
                            |
                 +----------+----------+
                 |                     |
              PRIMARY              SECONDARY
             Mumbai                 Hyderabad
           ap-south-1              ap-south-2
                 |                     |
        Route 53 Health Check   No secondary health
        hc-mumbai               check in this lab
                 |                     |
                 v                     v
        Mumbai ALB              Hyderabad ALB
        HTTP :80                HTTP :80
                 |                     |
                 v                     v
        Target Group            Target Group
                 |                     |
                 v                     v
          EC2 + Nginx            EC2 + Nginx
                 |                     |
        MUMBAI PRIMARY          HYDERABAD SECONDARY
```

## Failover behavior

Normal condition:

``` text
User
 |
 v
Route 53
 |
 | Primary Healthy
 v
Mumbai ALB
 |
 v
Mumbai EC2
 |
 v
Nginx
```

Failure condition:

``` text
Mumbai Nginx/EC2/Application fails
              |
              v
       hc-mumbai = Unhealthy
              |
              v
       Route 53 detects failure
              |
              v
       Secondary record selected
              |
              v
       Hyderabad ALB
              |
              v
       Hyderabad EC2
              |
              v
       Nginx
```

Recovery:

``` text
Mumbai Nginx restored
       |
       v
hc-mumbai = Healthy
       |
       v
Route 53 can select Primary again
       |
       v
Mumbai ALB
```

------------------------------------------------------------------------

# 3. Resources Created

## Account A -- Mumbai

  Resource           Value
  ------------------ -----------------------------------------------------------
  Region             `ap-south-1`
  VPC                `mum-prod-vpc`
  CIDR               `10.10.0.0/16`
  Public subnets     2
  Private subnets    2
  ALB                `mum-app-primary-1426974919`
  ALB DNS            `mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com`
  Health check       `hc-mumbai`
  Application        Nginx
  Application port   `80`

## Account B -- Hyderabad

  Resource           Value
  ------------------ ---------------------------------------------------
  Region             `ap-south-2`
  VPC                `hyd-dr`
  CIDR               `10.20.0.0/16`
  ALB                `hyd-alb-1968825055`
  ALB DNS            `hyd-alb-1968825055.ap-south-2.elb.amazonaws.com`
  Health endpoint    `/health`
  Application        Nginx
  Application port   `80`

## VPC Peering

  Item                 Value
  -------------------- -------------------------
  Peering connection   `pcx-00b5c95712b424a4b`
  Mumbai VPC           `10.10.0.0/16`
  Hyderabad VPC        `10.20.0.0/16`
  Status               ACTIVE

The peering connection was created between the two VPCs so that private
resources can communicate if required.

For this particular DNS failover path, the client does not traverse the
VPC peering connection. Internet traffic reaches the appropriate public
ALB directly.

------------------------------------------------------------------------

# 4. Prerequisites

Before starting:

-   AWS Account A
-   AWS Account B
-   Access to both AWS accounts
-   AWS Console access
-   One domain
-   Permission to modify domain nameservers
-   Route 53 access in Account A
-   EC2 permissions
-   VPC permissions
-   Elastic Load Balancing permissions
-   Route 53 Health Check permissions
-   Linux/SSH access to EC2
-   Nginx
-   `curl`
-   `dig`

Optional CLI verification:

``` bash
aws configure
```

Check AWS identity:

``` bash
aws sts get-caller-identity
```

Check AWS region:

``` bash
aws configure get region
```

------------------------------------------------------------------------

# 5. IP Address Planning

Use non-overlapping CIDR blocks.

## Mumbai

``` text
VPC: 10.10.0.0/16
```

## Hyderabad

``` text
VPC: 10.20.0.0/16
```

Because these CIDRs do not overlap, VPC peering is possible.

Example subnet design:

``` text
Mumbai VPC
10.10.0.0/16

Public Subnet AZ-1
10.10.1.0/24

Public Subnet AZ-2
10.10.2.0/24

Private Subnet AZ-1
10.10.11.0/24

Private Subnet AZ-2
10.10.12.0/24
```

The exact subnet IDs and AZ names can vary.

------------------------------------------------------------------------

# 6. Create Mumbai VPC

Log in to **Account A**.

Set region:

``` text
Mumbai
ap-south-1
```

Go to:

``` text
VPC Console
→ Your VPCs
→ Create VPC
```

Create:

``` text
Name: mum-prod-vpc
IPv4 CIDR: 10.10.0.0/16
```

------------------------------------------------------------------------

# 7. Create Mumbai Subnets

Create two public and two private subnets across Availability Zones.

Example:

``` text
Public-1
10.10.1.0/24

Public-2
10.10.2.0/24

Private-1
10.10.11.0/24

Private-2
10.10.12.0/24
```

The important requirement is:

-   Public subnets should be in different AZs.
-   Private subnets should be in different AZs.

------------------------------------------------------------------------

# 8. Internet Gateway

Create an Internet Gateway:

``` text
Name:
mum-prod-igw
```

Attach it to:

``` text
mum-prod-vpc
```

Public route table:

``` text
Destination: 0.0.0.0/0
Target: Internet Gateway
```

This allows public resources to communicate with the Internet.

------------------------------------------------------------------------

# 9. Mumbai EC2 Application Server

Launch an EC2 instance in the Mumbai VPC.

Example:

``` text
Region:
ap-south-1

VPC:
mum-prod-vpc

Subnet:
Public subnet

Auto-assign Public IP:
Enabled
```

Install Nginx.

For Ubuntu:

``` bash
sudo apt update
sudo apt install nginx -y
```

Check status:

``` bash
sudo systemctl status nginx
```

Start and enable:

``` bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

Test locally:

``` bash
curl http://localhost
```

------------------------------------------------------------------------

# 10. Configure Mumbai Application Page

Create a page identifying the primary region:

``` bash
sudo tee /var/www/html/index.html > /dev/null <<'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Mumbai Primary</title>
</head>
<body>
    <h1>MUMBAI PRIMARY SERVER</h1>
    <p>Account A - ap-south-1</p>
</body>
</html>
EOF
```

Test:

``` bash
curl http://localhost
```

Expected content:

``` text
MUMBAI PRIMARY SERVER
Account A - ap-south-1
```

------------------------------------------------------------------------

# 11. Configure Mumbai Health Endpoint

Route 53 must be able to determine whether the primary application is
healthy.

Create:

``` bash
sudo tee /var/www/html/health > /dev/null <<'EOF'
OK
EOF
```

Test locally:

``` bash
curl http://localhost/health
```

Expected:

``` text
OK
```

This endpoint is intentionally simple.

The health check does not test the complete application stack. It
verifies that the HTTP endpoint is responding.

------------------------------------------------------------------------

# 12. Mumbai EC2 Security Group

Allow HTTP:

``` text
Inbound:
TCP 80
Source:
ALB Security Group
```

For initial setup/testing, SSH can be allowed from your administrator
IP.

Example:

``` text
TCP 22
Source:
My IP
```

Do not unnecessarily expose SSH to:

``` text
0.0.0.0/0
```

------------------------------------------------------------------------

# 13. Create Mumbai Target Group

Go to:

``` text
EC2
→ Target Groups
→ Create target group
```

Select:

``` text
Target type:
Instances

Protocol:
HTTP

Port:
80

VPC:
mum-prod-vpc
```

Health check:

``` text
Protocol:
HTTP

Path:
/health
```

Register the Mumbai EC2 instance.

Confirm target state becomes:

``` text
Healthy
```

------------------------------------------------------------------------

# 14. Create Mumbai Application Load Balancer

Go to:

``` text
EC2
→ Load Balancers
→ Create Load Balancer
```

Select:

``` text
Application Load Balancer
```

Example:

``` text
Name:
mum-app-primary
```

Scheme:

``` text
Internet-facing
```

IP address type:

``` text
IPv4
```

Select the Mumbai VPC.

Select at least two public subnets in different AZs.

Listener:

``` text
HTTP :80
```

Default action:

``` text
Forward to Mumbai target group
```

Create the ALB.

After creation, note the DNS name:

``` text
mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com
```

------------------------------------------------------------------------

# 15. Mumbai ALB Security Group

The ALB security group should allow:

``` text
Inbound
HTTP
TCP 80
Source:
0.0.0.0/0
```

The EC2 security group should allow HTTP from the ALB security group.

This creates the intended flow:

``` text
Internet
   |
   v
ALB Security Group
   |
   v
ALB
   |
   v
EC2 Security Group
   |
   v
EC2
```

------------------------------------------------------------------------

# 16. Test Mumbai ALB

Run:

``` bash
curl -i http://mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com
```

Expected:

``` text
HTTP/1.1 200 OK
```

and:

``` text
MUMBAI PRIMARY SERVER
```

Test health:

``` bash
curl -i http://mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com/health
```

Expected:

``` text
HTTP/1.1 200 OK

OK
```

This test is important because Route 53 will use this endpoint.

------------------------------------------------------------------------

# 17. Create Hyderabad VPC

Switch to **Account B**.

Set region:

``` text
Hyderabad
ap-south-2
```

Create VPC:

``` text
Name:
hyd-dr

CIDR:
10.20.0.0/16
```

Create appropriate subnets.

For the ALB, use public subnets in different AZs.

------------------------------------------------------------------------

# 18. Hyderabad EC2 Application Server

Launch an EC2 instance in the Hyderabad VPC.

Install Nginx:

``` bash
sudo apt update
sudo apt install nginx -y
```

Enable and start:

``` bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

Test:

``` bash
curl http://localhost
```

------------------------------------------------------------------------

# 19. Configure Hyderabad Application Page

Create:

``` bash
sudo tee /var/www/html/index.html > /dev/null <<'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Hyderabad Secondary</title>
</head>
<body>
    <h1>HYDERABAD SECONDARY SERVER</h1>
    <p>Account B - ap-south-2</p>
</body>
</html>
EOF
```

Test:

``` bash
curl http://localhost
```

Expected:

``` text
HYDERABAD SECONDARY SERVER
Account B - ap-south-2
```

------------------------------------------------------------------------

# 20. Configure Hyderabad Health Endpoint

Create:

``` bash
sudo tee /var/www/html/health > /dev/null <<'EOF'
OK
EOF
```

Test:

``` bash
curl http://localhost/health
```

Expected:

``` text
OK
```

This endpoint is required because Route 53 will check the Hyderabad ALB.

------------------------------------------------------------------------

# 21. Hyderabad Target Group

Create an Application Load Balancer target group.

Example:

``` text
Target type:
Instances

Protocol:
HTTP

Port:
80

VPC:
hyd-dr

Health check path:
/health
```

Register the Hyderabad EC2 instance.

Confirm:

``` text
Healthy
```

------------------------------------------------------------------------

# 22. Hyderabad Application Load Balancer

Create:

``` text
Name:
hyd-alb
```

Scheme:

``` text
Internet-facing
```

Listener:

``` text
HTTP :80
```

Forward traffic to the Hyderabad target group.

The resulting DNS name used in this project is:

``` text
hyd-alb-1968825055.ap-south-2.elb.amazonaws.com
```

------------------------------------------------------------------------

# 23. Test Hyderabad ALB

Run:

``` bash
curl -i http://hyd-alb-1968825055.ap-south-2.elb.amazonaws.com
```

Expected:

``` text
HTTP/1.1 200 OK
```

and:

``` text
HYDERABAD SECONDARY SERVER
```

Health endpoint:

``` bash
curl -i http://hyd-alb-1968825055.ap-south-2.elb.amazonaws.com/health
```

Expected:

``` text
HTTP/1.1 200 OK

OK
```

------------------------------------------------------------------------

# 24. VPC Peering

The two VPCs have non-overlapping CIDRs:

``` text
Mumbai:
10.10.0.0/16

Hyderabad:
10.20.0.0/16
```

Create VPC peering.

The resulting peering connection:

``` text
pcx-00b5c95712b424a4b
```

Status:

``` text
ACTIVE
```

Mumbai:

``` text
Requester
10.10.0.0/16
```

Hyderabad:

``` text
Accepter
10.20.0.0/16
```

------------------------------------------------------------------------

# 25. Mumbai Route Table for Peering

In Mumbai private route tables, add:

``` text
Destination:
10.20.0.0/16

Target:
pcx-00b5c95712b424a4b
```

This means:

``` text
10.10.0.0/16
        |
        | VPC Peering
        v
10.20.0.0/16
```

The public route table was intentionally not given the peering route
because it is not required for the public ALB failover path.

------------------------------------------------------------------------

# 26. Hyderabad Route Table for Peering

If private communication is required in the reverse direction, the
Hyderabad private route table should contain:

``` text
Destination:
10.10.0.0/16

Target:
pcx-00b5c95712b424a4b
```

Important:

> VPC peering routing is not transitive.

The peering connection is not used as a general Internet path.

------------------------------------------------------------------------

# 27. Domain Registration

The domain used:

``` text
demodomain.fun
```

The domain was originally registered with:

``` text
Hostinger
```

The DNS zone is hosted in:

``` text
Amazon Route 53
```

The registrar and DNS hosting provider do not have to be the same.

------------------------------------------------------------------------

# 28. Create Route 53 Hosted Zone

In Account A:

``` text
Route 53
→ Hosted zones
→ Create hosted zone
```

Domain:

``` text
demodomain.fun
```

Type:

``` text
Public hosted zone
```

Route 53 generates four nameservers.

The nameservers used in this project were:

``` text
ns-1426.awsdns-50.org.
ns-1673.awsdns-17.co.uk.
ns-382.awsdns-47.com.
ns-928.awsdns-52.net.
```

------------------------------------------------------------------------

# 29. Update Hostinger Nameservers

At Hostinger, replace the existing nameservers with the Route 53
nameservers.

Use:

``` text
ns-1426.awsdns-50.org.
ns-1673.awsdns-17.co.uk.
ns-382.awsdns-47.com.
ns-928.awsdns-52.net.
```

DNS propagation can take time.

Verify:

``` bash
dig NS demodomain.fun
```

Or:

``` bash
dig NS demodomain.fun @1.1.1.1
```

Expected nameservers should be the AWS Route 53 nameservers.

------------------------------------------------------------------------

# 30. Route 53 Health Check -- Mumbai

Go to:

``` text
Route 53
→ Health checks
→ Create health check
```

Configure:

``` text
Name:
hc-mumbai

What to monitor:
Endpoint

Protocol:
HTTP

IP address:
Leave blank when using the hostname option

Domain name:
mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com

Port:
80

Path:
/health
```

In the console UI used during this lab, the endpoint information can
appear in the domain field in the form:

``` text
hostname:80/health
```

The important effective values are:

``` text
HTTP
Port 80
Path /health
```

After creation, wait for the health check to run.

Expected:

``` text
Healthy
```

------------------------------------------------------------------------

# 31. Route 53 Health Check -- Hyderabad

Create another health check:

``` text
Name:
hc-hyderabad

Protocol:
HTTP

Endpoint:
hyd-alb-1968825055.ap-south-2.elb.amazonaws.com

Port:
80

Path:
/health
```

Expected:

``` text
Healthy
```

------------------------------------------------------------------------

# 32. Create Primary Failover Record

Go to:

``` text
Route 53
→ Hosted zones
→ demodomain.fun
→ Create record
```

Use:

``` text
Record name:
app

Record type:
CNAME

Value:
mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com

TTL:
60

Routing policy:
Failover

Failover record type:
Primary

Health check:
hc-mumbai

Record ID:
mum-primary
```

Create the record.

The resulting FQDN is:

``` text
app.demodomain.fun
```

------------------------------------------------------------------------

# 33. Create Secondary Failover Record

Create another record with the same name:

``` text
Record name:
app
```

Type:

``` text
CNAME
```

Value:

``` text
hyd-alb-1968825055.ap-south-2.elb.amazonaws.com
```

TTL:

``` text
60
```

Routing policy:

``` text
Failover
```

Failover record type:

``` text
Secondary
```

Record ID:

``` text
hyd-secondary
```

### Important configuration used in this lab

Leave the Secondary health check **blank**.

So:

``` text
Primary:
hc-mumbai

Secondary:
No health check
```

This makes the secondary the deterministic fallback when the primary is
unhealthy.

------------------------------------------------------------------------

# 34. Final Route 53 Configuration

The hosted zone contains four records:

``` text
NS
SOA
app.demodomain.fun – Primary
app.demodomain.fun – Secondary
```

The application records are:

### Primary

``` text
Name:
app.demodomain.fun

Type:
CNAME

Value:
mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com

TTL:
60

Routing:
Failover / Primary

Health check:
hc-mumbai

Record ID:
mum-primary
```

### Secondary

``` text
Name:
app.demodomain.fun

Type:
CNAME

Value:
hyd-alb-1968825055.ap-south-2.elb.amazonaws.com

TTL:
60

Routing:
Failover / Secondary

Health check:
None

Record ID:
hyd-secondary
```

------------------------------------------------------------------------

# 35. Test DNS Before Failure

Make sure Mumbai is healthy.

Check:

``` bash
dig @1.1.1.1 +short app.demodomain.fun
```

Expected:

``` text
mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com.
```

It can also return the ALB IP addresses after DNS resolution, for
example:

``` text
13.232.164.235
13.205.103.25
```

The exact ALB IP addresses can change.

------------------------------------------------------------------------

# 36. Test Application Before Failure

Run:

``` bash
curl -i http://app.demodomain.fun
```

Expected:

``` text
HTTP/1.1 200 OK
```

Response:

``` text
MUMBAI PRIMARY SERVER
Account A - ap-south-1
```

Health:

``` bash
curl -i http://app.demodomain.fun/health
```

Expected:

``` text
HTTP/1.1 200 OK

OK
```

------------------------------------------------------------------------

# 37. Confirm Route 53 Health Checks

Open:

``` text
Route 53
→ Health checks
```

Expected:

``` text
hc-mumbai       Healthy
hc-hyderabad    Healthy
```

At this stage, traffic should go to Mumbai because Mumbai is the Primary
record.

------------------------------------------------------------------------

# 38. Perform the Failover Test

This is the most important test.

Simulate Mumbai application failure.

On the Mumbai EC2 instance:

``` bash
sudo systemctl stop nginx
```

Do not stop the ALB.

The ALB will still exist, but its target will stop responding to HTTP.

------------------------------------------------------------------------

# 39. Confirm Mumbai Application Failure

Test the Mumbai ALB directly:

``` bash
curl -i http://mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com
```

The ALB may return:

``` text
502 Bad Gateway
```

This is expected in this test because the ALB cannot successfully reach
the stopped Nginx service.

Test:

``` bash
curl -i http://mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com/health
```

It should no longer return the expected `200 OK`.

------------------------------------------------------------------------

# 40. Wait for Route 53 Health Check Failure

Open:

``` text
Route 53
→ Health checks
```

Expected:

``` text
hc-mumbai       Unhealthy
hc-hyderabad    Healthy
```

The exact time to detect a failure depends on the Route 53 health-check
configuration and its checkers.

------------------------------------------------------------------------

# 41. Verify DNS Failover

Run:

``` bash
dig @1.1.1.1 +short app.demodomain.fun
```

Before failure:

``` text
mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com.
```

After failover:

``` text
hyd-alb-1968825055.ap-south-2.elb.amazonaws.com.
```

During the successful test, the Hyderabad ALB resolved to IP addresses
such as:

``` text
16.113.61.1
40.192.22.9
```

The actual ALB IPs can change and should not be hard-coded.

------------------------------------------------------------------------

# 42. Verify Application Failover

Run:

``` bash
curl -i http://app.demodomain.fun
```

Expected:

``` text
HTTP/1.1 200 OK
```

Application response:

``` text
HYDERABAD SECONDARY SERVER
Account B - ap-south-2
```

Health endpoint:

``` bash
curl -i http://app.demodomain.fun/health
```

Expected:

``` text
HTTP/1.1 200 OK

OK
```

This confirms the complete chain:

``` text
Mumbai failure
      ↓
Route 53 health check detects failure
      ↓
Primary marked unhealthy
      ↓
Route 53 returns Secondary record
      ↓
DNS resolves to Hyderabad ALB
      ↓
Hyderabad EC2
      ↓
Nginx
      ↓
HTTP 200
```

------------------------------------------------------------------------

# 43. Why the First Failover Attempt Showed Mumbai

During the troubleshooting process, Mumbai Nginx was stopped, but DNS
initially continued returning the Mumbai ALB.

The sequence was:

``` text
Mumbai Nginx stopped
        ↓
Mumbai ALB returned 502
        ↓
hc-mumbai became Unhealthy
        ↓
DNS resolver still returned Mumbai
```

The important distinction is:

> Route 53 health-check state and cached DNS answers are separate
> things.

A recursive resolver can continue serving a previously cached answer
until the TTL expires.

The record TTL used in this lab was:

``` text
60 seconds
```

After the Route 53 failover state updated and the resolver refreshed its
cached answer, DNS returned the Hyderabad ALB.

------------------------------------------------------------------------

# 44. Authoritative DNS Troubleshooting

If a public resolver appears to return an old answer, query a Route 53
authoritative nameserver directly.

Example:

``` bash
dig @ns-1673.awsdns-17.co.uk app.demodomain.fun
```

This helps distinguish:

``` text
Route 53 authoritative answer
```

from:

``` text
Recursive resolver cache
```

Useful recursive resolver:

``` bash
dig @1.1.1.1 +short app.demodomain.fun
```

------------------------------------------------------------------------

# 45. Failback Test

After confirming Hyderabad is serving traffic, restore Mumbai.

On the Mumbai EC2:

``` bash
sudo systemctl start nginx
```

Verify:

``` bash
sudo systemctl status nginx
```

Test locally:

``` bash
curl http://localhost/health
```

Expected:

``` text
OK
```

Test the Mumbai ALB:

``` bash
curl -i http://mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com/health
```

Expected:

``` text
HTTP/1.1 200 OK
```

------------------------------------------------------------------------

# 46. Wait for Mumbai Health Recovery

Route 53 should eventually show:

``` text
hc-mumbai       Healthy
```

Hyderabad should remain:

``` text
hc-hyderabad    Healthy
```

Then query:

``` bash
dig @1.1.1.1 +short app.demodomain.fun
```

The answer should eventually return:

``` text
mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com.
```

Test:

``` bash
curl -i http://app.demodomain.fun
```

Expected:

``` text
MUMBAI PRIMARY SERVER
Account A - ap-south-1
```

This confirms failback.

------------------------------------------------------------------------

# 47. Complete End-to-End Test Matrix

  Test                         Expected Result
  ---------------------------- ---------------------------------
  Mumbai EC2 → Nginx           200
  Mumbai `/health`             `OK`
  Mumbai ALB                   200
  Mumbai ALB `/health`         200
  Hyderabad EC2 → Nginx        200
  Hyderabad `/health`          `OK`
  Hyderabad ALB                200
  Hyderabad ALB `/health`      200
  Both health checks healthy   Primary selected
  Stop Mumbai Nginx            Mumbai health becomes unhealthy
  DNS after failover           Hyderabad ALB
  Application after failover   Hyderabad page
  Start Mumbai Nginx           Mumbai health recovers
  DNS after recovery           Mumbai ALB
  Application after recovery   Mumbai page

------------------------------------------------------------------------

# 48. Useful Verification Commands

## DNS

``` bash
dig app.demodomain.fun
```

Short answer:

``` bash
dig +short app.demodomain.fun
```

Use Cloudflare DNS:

``` bash
dig @1.1.1.1 +short app.demodomain.fun
```

Use Google DNS:

``` bash
dig @8.8.8.8 +short app.demodomain.fun
```

Query an authoritative nameserver:

``` bash
dig @ns-1673.awsdns-17.co.uk app.demodomain.fun
```

------------------------------------------------------------------------

## HTTP

Mumbai:

``` bash
curl -i http://mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com
```

Mumbai health:

``` bash
curl -i http://mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com/health
```

Hyderabad:

``` bash
curl -i http://hyd-alb-1968825055.ap-south-2.elb.amazonaws.com
```

Hyderabad health:

``` bash
curl -i http://hyd-alb-1968825055.ap-south-2.elb.amazonaws.com/health
```

Application DNS:

``` bash
curl -i http://app.demodomain.fun
```

Application health:

``` bash
curl -i http://app.demodomain.fun/health
```

------------------------------------------------------------------------

## Nginx

Check:

``` bash
sudo systemctl status nginx
```

Stop:

``` bash
sudo systemctl stop nginx
```

Start:

``` bash
sudo systemctl start nginx
```

Restart:

``` bash
sudo systemctl restart nginx
```

Enable at boot:

``` bash
sudo systemctl enable nginx
```

------------------------------------------------------------------------

# 49. Security Group Design

## ALB Security Group

Inbound:

``` text
HTTP
TCP 80
Source: 0.0.0.0/0
```

Outbound:

``` text
Allow required traffic to targets
```

## EC2 Security Group

Inbound:

``` text
HTTP
TCP 80
Source: ALB Security Group
```

SSH:

``` text
TCP 22
Source: Administrator/My IP
```

This is preferable to exposing the EC2 HTTP service directly to the
entire Internet.

------------------------------------------------------------------------

# 50. Why ALB Is Used

The ALB provides:

-   Layer 7 HTTP load balancing
-   Health checks
-   Target registration
-   Multi-AZ deployment
-   Stable DNS endpoint
-   Traffic forwarding to EC2 instances

The client does not directly connect to the EC2 instance.

The flow is:

``` text
Client
 ↓
Route 53
 ↓
ALB
 ↓
Target Group
 ↓
EC2
 ↓
Nginx
```

------------------------------------------------------------------------

# 51. Why Route 53 Health Checks Are Used

The ALB's own target health determines whether the ALB can route to its
targets.

The Route 53 health check provides an additional DNS-level decision.

In this project:

``` text
Route 53
    |
    | HTTP /health
    v
Mumbai ALB
    |
    v
Mumbai Nginx
```

If the endpoint fails, Route 53 can mark the primary unhealthy and
select the secondary failover record.

------------------------------------------------------------------------

# 52. Why the Secondary Health Check Was Removed

The final configuration intentionally uses:

``` text
Primary:
Health check = hc-mumbai

Secondary:
Health check = none
```

For this active-passive lab, the objective is:

``` text
If Primary is unhealthy
        ↓
always use Secondary
```

If a secondary health check is also configured, Route 53's failover
selection can depend on the health state of both records.

Removing the Secondary health check makes the fallback behavior simpler
and deterministic for this lab.

This does not mean that a secondary health check is universally wrong.
In a production design, the health-check strategy should match the
desired failure semantics.

------------------------------------------------------------------------

# 53. Why CNAME Is Used

The records are:

``` text
app.demodomain.fun
        |
        v
CNAME
        |
        v
AWS ALB DNS name
```

Example:

``` text
app.demodomain.fun
→ mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com
```

and:

``` text
app.demodomain.fun
→ hyd-alb-1968825055.ap-south-2.elb.amazonaws.com
```

ALB IP addresses should not be hard-coded because AWS manages the load
balancer's underlying IP addresses.

------------------------------------------------------------------------

# 54. Why TTL Is 60 Seconds

The records use:

``` text
TTL = 60 seconds
```

A shorter TTL helps clients and recursive DNS resolvers refresh the
answer sooner.

However:

> TTL does not mean failover always completes in exactly 60 seconds.

Actual failover time also depends on:

-   Route 53 health-check detection
-   Health-check configuration
-   Resolver behavior
-   Existing DNS cache
-   Application/client caching

------------------------------------------------------------------------

# 55. Important Difference: DNS Failover vs Application Failover

Route 53 does not move the application.

It changes the DNS answer.

Before failure:

``` text
app.demodomain.fun
        ↓
Mumbai ALB
```

After failure:

``` text
app.demodomain.fun
        ↓
Hyderabad ALB
```

The Hyderabad environment must already be running and ready to serve
traffic.

This is why the secondary application was deployed before performing the
failure test.

------------------------------------------------------------------------

# 56. What Happens If Mumbai ALB Fails

The health-check path is:

``` text
Route 53
 ↓
Mumbai ALB
 ↓
Mumbai target
 ↓
Nginx
```

If the application cannot respond successfully:

``` text
hc-mumbai = Unhealthy
```

Route 53 then selects the Secondary failover record.

------------------------------------------------------------------------

# 57. What Happens If Hyderabad Is Also Down

In this specific configuration, the Secondary record has no health
check.

Therefore, when the Primary is unhealthy, Route 53 can return the
Secondary record even if the secondary application itself is not
responding.

This is a deliberate simplification of the lab.

For a production architecture, the secondary environment should have its
own health validation and operational monitoring strategy.

------------------------------------------------------------------------

# 58. Database/RDS Scope

A database was considered during the design, including:

``` text
DB subnet group
```

and:

``` text
Parameter group:
mum-mysql-dms
Engine:
MySQL 8.0
binlog_format:
ROW
```

However, an actual RDS database was intentionally **not deployed** for
this lab.

Therefore this project demonstrates:

``` text
Application infrastructure failover
```

but not:

``` text
Database failover
```

There is currently no database replication mechanism between Mumbai and
Hyderabad.

------------------------------------------------------------------------

# 59. HTTPS Scope

This lab uses:

``` text
HTTP :80
```

It does not use:

-   ACM
-   HTTPS
-   TLS certificates
-   HTTP → HTTPS redirects

For production, HTTPS should be added with certificates managed through
AWS Certificate Manager.

------------------------------------------------------------------------

# 60. WAF Scope

AWS WAF was intentionally excluded from the final lab.

Therefore:

``` text
Internet
 ↓
Route 53
 ↓
ALB
```

There is no WAF layer in this implementation.

A future production enhancement could place AWS WAF in front of the
application load balancers.

------------------------------------------------------------------------

# 61. Current Architecture Limitations

## 1. No database replication

The application can fail over, but database state is not replicated.

If the application depends on a database, the secondary application may
not have the latest data.

------------------------------------------------------------------------

## 2. DNS failover is not instantaneous

DNS caching means clients may temporarily continue using the previous
DNS answer.

The TTL is only one factor in the total failover time.

------------------------------------------------------------------------

## 3. Secondary health check is not configured

The secondary is treated as the fallback destination.

If Hyderabad is down at the same time, Route 53 can still return the
secondary record.

------------------------------------------------------------------------

## 4. No HTTPS

Traffic is HTTP.

Credentials, sessions, and application data should not be sent over
unencrypted HTTP in production.

------------------------------------------------------------------------

## 5. No WAF

The ALB is directly exposed to Internet traffic.

There is no AWS WAF protection in this lab.

------------------------------------------------------------------------

## 6. Manual application deployment

The application environments are manually configured.

There is no:

-   CI/CD pipeline
-   Infrastructure as Code
-   Automated deployment
-   Automated configuration management

------------------------------------------------------------------------

## 7. Single EC2 application server per environment

Although the ALB is multi-AZ, this lab's application deployment is
simplified around the EC2/Nginx setup.

A production application should normally have multiple application
targets distributed across AZs.

------------------------------------------------------------------------

## 8. No automated infrastructure recovery

If the EC2 instance itself is lost, there is no automated cross-region
infrastructure rebuild in this lab.

------------------------------------------------------------------------

## 9. No centralized observability

The lab does not currently include a complete:

-   CloudWatch dashboard
-   centralized logging
-   alerting
-   incident management
-   synthetic monitoring

setup.

------------------------------------------------------------------------

## 10. No automated failover orchestration

Route 53 performs DNS failover, but there is no workflow that
automatically:

-   rebuilds infrastructure
-   synchronizes data
-   deploys the latest application
-   validates the DR environment

------------------------------------------------------------------------

# 62. Production Enhancements

The next version could add:

``` text
                    Route 53
                       |
                 Global DNS
                       |
          +------------+------------+
          |                         |
       Mumbai                    Hyderabad
       Primary                   Secondary
          |                         |
        WAF                       WAF
          |                         |
        ALB                       ALB
          |                         |
       ASG/EKS                  ASG/EKS
          |                         |
      Application              Application
          |                         |
          +-----------+-------------+
                      |
              Database Layer
```

Potential enhancements:

-   AWS WAF
-   HTTPS with ACM
-   Auto Scaling Groups
-   Multiple EC2 instances per AZ
-   RDS
-   Cross-region database strategy
-   AWS Backup
-   S3 replication
-   CloudWatch monitoring
-   CloudWatch alarms
-   SNS notifications
-   Infrastructure as Code with Terraform
-   CI/CD
-   Systems Manager
-   centralized logging
-   automated DR testing
-   Route 53 Resolver where private DNS is required
-   AWS Global Accelerator for use cases where DNS-based failover is
    insufficient

------------------------------------------------------------------------

# 63. Interview Explanation

A concise explanation of the project:

> I built an active-passive cross-region application failover
> architecture across two AWS accounts. The primary application runs in
> Mumbai in `ap-south-1`, and the secondary application runs in
> Hyderabad in `ap-south-2`. Each environment has an EC2 instance
> running Nginx behind an internet-facing Application Load Balancer.
> Route 53 provides failover routing using a primary health check
> against `/health`. If the Mumbai application becomes unhealthy, Route
> 53 changes the DNS response from the Mumbai ALB to the Hyderabad ALB.
> I tested the architecture by stopping Nginx on the Mumbai instance,
> verifying that the Mumbai health check became unhealthy, confirming
> that DNS resolved to the Hyderabad ALB, and validating that the
> application returned the Hyderabad response. I then restored Mumbai
> and verified failback.

------------------------------------------------------------------------

# 64. Interview Questions

## Q1. Why did you use Route 53 failover routing?

Because the requirement was active-passive DNS-based failover between
two regional application environments.

## Q2. What happens when Mumbai fails?

The Route 53 health check marks Mumbai unhealthy and Route 53 returns
the Secondary Hyderabad record.

## Q3. Does Route 53 move traffic directly?

No. Route 53 changes the DNS response. The client then connects to the
ALB returned by DNS.

## Q4. Why use an ALB?

The ALB provides Layer 7 HTTP load balancing, health checks, target
management and a stable AWS-managed DNS endpoint.

## Q5. Why use `/health`?

It provides a simple endpoint that Route 53 can check to determine
whether the application is responding.

## Q6. Why are the VPC CIDRs different?

VPC peering requires non-overlapping address ranges for normal routed
communication.

## Q7. Is VPC peering required for DNS failover?

No.

The DNS failover path is:

``` text
Client
 ↓
Route 53
 ↓
ALB
```

VPC peering is a separate network connectivity component.

## Q8. Why is the Secondary health check blank?

For this lab, the Secondary is intended to be the deterministic fallback
when the Primary becomes unhealthy.

## Q9. Does the 60-second TTL guarantee 60-second failover?

No.

The total time also depends on Route 53 health-check detection and DNS
resolver caching.

## Q10. What is missing for a real DR solution?

The major missing component is data protection and replication.

The current lab does not implement database replication.

------------------------------------------------------------------------

# 65. Final Architecture Summary

``` text
                           INTERNET
                               |
                               v
                     app.demodomain.fun
                               |
                               v
                     AMAZON ROUTE 53
                     FAILOVER ROUTING
                               |
                +--------------+--------------+
                |                             |
             PRIMARY                       SECONDARY
             Mumbai                        Hyderabad
           ap-south-1                    ap-south-2
                |                             |
        hc-mumbai                         No HC
                |                             |
                v                             v
          Mumbai ALB                  Hyderabad ALB
                |                             |
                v                             v
          Target Group                 Target Group
                |                             |
                v                             v
          EC2 + Nginx                   EC2 + Nginx
                |                             |
                v                             v
       MUMBAI PRIMARY             HYDERABAD SECONDARY
```

### Normal state

``` text
app.demodomain.fun
        ↓
Mumbai ALB
        ↓
Mumbai EC2
        ↓
Nginx
```

### Failure state

``` text
Mumbai Nginx fails
        ↓
hc-mumbai = Unhealthy
        ↓
Route 53 selects Secondary
        ↓
Hyderabad ALB
        ↓
Hyderabad EC2
        ↓
Nginx
```

### Recovery state

``` text
Mumbai Nginx restored
        ↓
hc-mumbai = Healthy
        ↓
Route 53 selects Primary
        ↓
Mumbai ALB
        ↓
Mumbai EC2
        ↓
Nginx
```

------------------------------------------------------------------------

# 66. Final Validation Checklist

Before considering the project complete:

-   [x] Mumbai VPC created
-   [x] Hyderabad VPC created
-   [x] CIDRs do not overlap
-   [x] Mumbai subnets created
-   [x] Hyderabad subnets created
-   [x] Internet connectivity configured
-   [x] Mumbai EC2 created
-   [x] Hyderabad EC2 created
-   [x] Nginx installed on Mumbai
-   [x] Nginx installed on Hyderabad
-   [x] Mumbai application page configured
-   [x] Hyderabad application page configured
-   [x] `/health` created on Mumbai
-   [x] `/health` created on Hyderabad
-   [x] Mumbai ALB created
-   [x] Hyderabad ALB created
-   [x] ALB target groups configured
-   [x] ALB health checks verified
-   [x] Security groups configured
-   [x] VPC peering created
-   [x] VPC peering is ACTIVE
-   [x] Mumbai private route table has Hyderabad CIDR route
-   [x] Route 53 public hosted zone created
-   [x] Hostinger nameservers changed to Route 53
-   [x] Route 53 nameserver propagation verified
-   [x] Mumbai Route 53 health check created
-   [x] Hyderabad Route 53 health check created
-   [x] Primary failover record created
-   [x] Secondary failover record created
-   [x] Primary health check attached
-   [x] Secondary health check left blank
-   [x] TTL set to 60 seconds
-   [x] Primary DNS resolution verified
-   [x] Primary application verified
-   [x] Mumbai failure simulated
-   [x] Mumbai health check became unhealthy
-   [x] DNS switched to Hyderabad
-   [x] Hyderabad application verified
-   [x] Mumbai recovered
-   [x] Mumbai health check recovered
-   [x] DNS failback verified
-   [x] Mumbai application verified again

------------------------------------------------------------------------

# 67. Cleanup

To avoid unnecessary AWS charges, delete resources when the lab is no
longer required.

Recommended cleanup order:

1.  Delete Route 53 application failover records.
2.  Delete Route 53 health checks.
3.  Delete Route 53 hosted zone if the domain will no longer use it.
4.  Restore/change nameservers at the registrar if necessary.
5.  Delete Hyderabad ALB.
6.  Delete Hyderabad target group.
7.  Terminate Hyderabad EC2.
8.  Delete Mumbai ALB.
9.  Delete Mumbai target group.
10. Terminate Mumbai EC2.
11. Delete unused security groups.
12. Delete VPC peering.
13. Delete unused route table entries.
14. Delete subnets.
15. Detach/delete Internet Gateways.
16. Delete VPCs.

Before deleting the hosted zone, make sure the domain is no longer
depending on its DNS records.

------------------------------------------------------------------------

# 68. Project Outcome

This project demonstrates an end-to-end AWS high-availability pattern
using:

``` text
AWS VPC
+
EC2
+
Nginx
+
Application Load Balancer
+
Route 53 Health Checks
+
Route 53 Failover Routing
+
Two AWS Accounts
+
Two AWS Regions
+
VPC Peering
```

The final tested behavior is:

``` text
                    NORMAL
                       |
                       v
               Mumbai Primary
                       |
                Mumbai ALB
                       |
                 Mumbai EC2
                       |
                    Nginx


                    FAILURE
                       |
                       v
             Mumbai becomes unhealthy
                       |
                       v
                Route 53 Failover
                       |
                       v
             Hyderabad Secondary
                       |
                Hyderabad ALB
                       |
                 Hyderabad EC2
                       |
                    Nginx


                    RECOVERY
                       |
                       v
             Mumbai becomes healthy
                       |
                       v
                Route 53 Failback
                       |
                       v
               Mumbai Primary
```

The hands-on test successfully demonstrated both:

``` text
FAILOVER
Mumbai → Hyderabad
```

and:

``` text
FAILBACK
Hyderabad → Mumbai
```


---

# 69. Exact Console Build Details That Must Not Be Skipped

The main lab describes the architecture and values. The following section fills in the execution details that are easy to miss when rebuilding it from scratch.

## 69.1 Mumbai VPC – Console Build

Go to:

```text
VPC Console
→ Your VPCs
→ Create VPC
```

Select:

```text
Resources to create:
VPC only
Name:
mum-prod-vpc
IPv4 CIDR:
10.10.0.0/16
IPv6:
No IPv6 CIDR block
Tenancy:
Default
```

Create the VPC.

## 69.2 Mumbai Subnets – Exact Plan

Create:

```text
Public subnet 1:
10.10.1.0/24

Public subnet 2:
10.10.2.0/24

Private subnet 1:
10.10.11.0/24

Private subnet 2:
10.10.12.0/24
```

Place the two public subnets in different Availability Zones.

Place the two private subnets in different Availability Zones.

Enable auto-assign public IPv4 addresses on the public subnet(s) that will host directly reachable test EC2 resources.

Do not rely on the default VPC.

## 69.3 Mumbai Route Tables – Exact Attachments

Create a public route table associated with the two public subnets.

Add:

```text
0.0.0.0/0
→ mum-prod-igw
```

If private resources need Internet egress, use the appropriate NAT architecture. This particular lab does not require a NAT Gateway for the public-EC2 application path, so do not create one only for the failover test.

Create/associate the private route tables with the private subnets.

## 69.4 Mumbai Security Groups – Create Before EC2/ALB Testing

Create an ALB security group:

```text
Name:
mum-alb-sg
```

Inbound:

```text
HTTP
TCP 80
Source:
0.0.0.0/0
```

Create an EC2 security group:

```text
Name:
mum-ec2-sg
```

Inbound:

```text
HTTP
TCP 80
Source:
mum-alb-sg
```

SSH:

```text
TCP 22
Source:
My IP
```

Use the ALB security group as the HTTP source to the EC2 security group. This prevents direct public HTTP access to the instance when the instance is reachable from the Internet.

## 69.5 Mumbai EC2 – Launch Checklist

Go to:

```text
EC2
→ Instances
→ Launch instance
```

Choose:

```text
AMI:
Ubuntu 22.04 (or the Ubuntu version you are using)

Instance type:
Choose a lab-appropriate low-cost instance

Key pair:
Your SSH key

VPC:
mum-prod-vpc

Subnet:
Mumbai application subnet

Auto-assign public IP:
Enabled only when using a directly reachable public EC2 for this lab

Security group:
mum-ec2-sg
```

Launch the instance.

Connect using your normal SSH method.

Verify:

```bash
curl http://localhost
```

Then install and configure Nginx using the steps in Sections 9–11.

## 69.6 Mumbai Target Group – Complete Values

Create the target group before creating the ALB.

```text
Target type:
Instances

Protocol:
HTTP

Port:
80

VPC:
mum-prod-vpc

Health check protocol:
HTTP

Health check path:
/health

Health check port:
Traffic port

Healthy threshold:
Use the console default unless you intentionally change it

Unhealthy threshold:
Use the console default unless you intentionally change it

Timeout:
Use the console default unless you intentionally change it

Interval:
Use the console default unless you intentionally change it

Success codes:
200
```

Register the Mumbai EC2 instance.

Do not continue until the target becomes:

```text
Healthy
```

## 69.7 Mumbai ALB – Complete Creation Checklist

Create:

```text
Type:
Application Load Balancer

Name:
mum-app-primary

Scheme:
Internet-facing

IP address type:
IPv4

VPC:
mum-prod-vpc

Subnets:
Two public subnets in different AZs

Security group:
mum-alb-sg

Listener:
HTTP :80

Default action:
Forward to the Mumbai target group
```

After deployment, copy the ALB DNS name exactly. In this completed lab it was:

```text
mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com
```

Never substitute an EC2 public IP for the ALB DNS name in Route 53.

## 69.8 Mumbai Pre-DNS Validation

Run all of these before creating Route 53 records:

```bash
curl -i http://localhost
curl -i http://localhost/health
curl -i http://mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com
curl -i http://mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com/health
```

You need:

```text
Local Nginx:
200

Local /health:
OK

Mumbai ALB:
200

Mumbai ALB /health:
200
```

Only then move to the Hyderabad build.

## 69.9 Hyderabad VPC – Console Build

In **Account B**, switch the region to:

```text
ap-south-2
```

Create:

```text
Name:
hyd-dr

IPv4 CIDR:
10.20.0.0/16
```

Create the public subnets required by the Internet-facing ALB in different AZs.

A matching four-subnet design can be:

```text
Public subnet 1:
10.20.1.0/24

Public subnet 2:
10.20.2.0/24

Private subnet 1:
10.20.11.0/24

Private subnet 2:
10.20.12.0/24
```

The exact subnet CIDRs can vary as long as they stay inside the VPC and do not overlap Mumbai.

## 69.10 Hyderabad Internet Gateway and Route Tables

Create and attach an Internet Gateway:

```text
hyd-dr-igw
```

Create a public route table and associate the public subnets.

Add:

```text
0.0.0.0/0
→ hyd-dr-igw
```

Create/associate private route tables with private subnets when using them.

## 69.11 Hyderabad Security Groups

Create an ALB security group:

```text
hyd-alb-sg
```

Inbound:

```text
HTTP
TCP 80
Source:
0.0.0.0/0
```

Create an EC2 security group:

```text
hyd-ec2-sg
```

Inbound:

```text
HTTP
TCP 80
Source:
hyd-alb-sg
```

SSH:

```text
TCP 22
Source:
My IP
```

## 69.12 Hyderabad EC2 – Launch Checklist

Use:

```text
VPC:
hyd-dr

Subnet:
Application subnet

Auto-assign public IP:
Enabled only if using directly reachable public EC2 for this lab

Security group:
hyd-ec2-sg
```

Install Nginx and create the Hyderabad page and `/health` endpoint.

Verify:

```bash
curl -i http://localhost
curl -i http://localhost/health
```

## 69.13 Hyderabad Target Group and ALB

Target group:

```text
Target type:
Instances

Protocol:
HTTP

Port:
80

VPC:
hyd-dr

Health path:
/health

Success code:
200
```

Then create the ALB:

```text
Name:
hyd-alb

Scheme:
Internet-facing

IP address type:
IPv4

VPC:
hyd-dr

Subnets:
Two public subnets in different AZs

Listener:
HTTP :80

Default action:
Forward to Hyderabad target group
```

The completed lab produced:

```text
hyd-alb-1968825055.ap-south-2.elb.amazonaws.com
```

Do not continue until:

```bash
curl -i http://hyd-alb-1968825055.ap-south-2.elb.amazonaws.com
curl -i http://hyd-alb-1968825055.ap-south-2.elb.amazonaws.com/health
```

both succeed.

## 69.14 Cross-Account VPC Peering – Exact Sequence

VPC peering is separate from Route 53 failover. It is included because this project also established private network connectivity between the two VPCs.

### Step 1 – Create request in Account A

In Account A:

```text
VPC
→ Peering connections
→ Create peering connection
```

Configure:

```text
Requester VPC:
mum-prod-vpc

Accepter account:
Another account

Accepter account ID:
Account B ID

Accepter VPC:
hyd-dr

Accepter region:
ap-south-2
```

Create the peering request.

### Step 2 – Accept in Account B

In Account B:

```text
VPC
→ Peering connections
```

Select the pending request.

Choose:

```text
Actions
→ Accept request
```

Confirm the status becomes:

```text
Active
```

Completed lab:

```text
pcx-00b5c95712b424a4b
```

### Step 3 – Add Mumbai-side route

For every Mumbai route table that must reach Hyderabad private resources:

```text
Destination:
10.20.0.0/16

Target:
pcx-00b5c95712b424a4b
```

The completed lab specifically added the route to the Mumbai private route tables.

### Step 4 – Add Hyderabad-side return route

For every Hyderabad route table that must return traffic to Mumbai private resources:

```text
Destination:
10.10.0.0/16

Target:
pcx-00b5c95712b424a4b
```

Without the required return route, traffic is not bidirectional.

### Step 5 – Verify

Confirm:

```text
Peering status:
Active

Mumbai route:
10.20.0.0/16 → pcx-00b5c95712b424a4b

Hyderabad return route:
10.10.0.0/16 → pcx-00b5c95712b424a4b
```

Again, the public DNS failover path does not depend on VPC peering.

## 69.15 Route 53 Delegation – Exact Final State

In Account A:

```text
Route 53
→ Hosted zones
→ demodomain.fun
```

There should be:

```text
NS
SOA
app.demodomain.fun – Primary
app.demodomain.fun – Secondary
```

The four AWS nameservers used in this completed lab were:

```text
ns-1426.awsdns-50.org.
ns-1673.awsdns-17.co.uk.
ns-382.awsdns-47.com.
ns-928.awsdns-52.net.
```

At Hostinger, replace the domain's existing nameservers with those four Route 53 nameservers.

Verify from a terminal:

```bash
dig NS demodomain.fun
dig NS demodomain.fun @1.1.1.1
```

The returned delegation must match the Route 53 hosted zone.

## 69.16 Route 53 Health Checks – Exact Test Before Records

Before creating failover records, directly verify:

```bash
curl -i http://mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com/health
curl -i http://hyd-alb-1968825055.ap-south-2.elb.amazonaws.com/health
```

Expected:

```text
HTTP 200
OK
```

Then create:

```text
hc-mumbai
hc-hyderabad
```

Make sure both show:

```text
Healthy
```

before testing the normal state.

## 69.17 Primary Record – Exact Final Form

Use:

```text
Name:
app

Type:
CNAME

TTL:
60

Routing policy:
Failover

Failover type:
Primary

Value:
mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com

Health check:
hc-mumbai

Record ID:
mum-primary
```

## 69.18 Secondary Record – Exact Final Form

Use:

```text
Name:
app

Type:
CNAME

TTL:
60

Routing policy:
Failover

Failover type:
Secondary

Value:
hyd-alb-1968825055.ap-south-2.elb.amazonaws.com

Health check:
Blank

Record ID:
hyd-secondary
```

The two records must have:

```text
Same record name
Same record type
Same routing policy
Unique record IDs
```

Only the failover role, value, and Primary health-check attachment differ.

---

# 70. Final No-Skip Validation Before Failure Test

Do not simulate failure until all of these are true:

```text
[ ] Account A = ap-south-1
[ ] Account B = ap-south-2

[ ] Mumbai VPC exists
[ ] Hyderabad VPC exists
[ ] CIDRs do not overlap

[ ] Mumbai public subnets exist in different AZs
[ ] Hyderabad public subnets exist in different AZs

[ ] Mumbai Internet Gateway attached
[ ] Hyderabad Internet Gateway attached

[ ] Mumbai ALB SG allows TCP 80 from Internet
[ ] Mumbai EC2 SG allows TCP 80 from Mumbai ALB SG
[ ] Hyderabad ALB SG allows TCP 80 from Internet
[ ] Hyderabad EC2 SG allows TCP 80 from Hyderabad ALB SG

[ ] Mumbai Nginx is running
[ ] Hyderabad Nginx is running

[ ] Mumbai /health = OK
[ ] Hyderabad /health = OK

[ ] Mumbai target group = Healthy
[ ] Hyderabad target group = Healthy

[ ] Mumbai ALB returns 200
[ ] Hyderabad ALB returns 200

[ ] VPC peering = Active
[ ] Required peering routes exist

[ ] Route 53 hosted zone exists
[ ] Hostinger delegates to Route 53
[ ] NS lookup shows Route 53 nameservers

[ ] hc-mumbai = Healthy
[ ] hc-hyderabad = Healthy

[ ] Primary record = Mumbai
[ ] Primary has hc-mumbai
[ ] Secondary record = Hyderabad
[ ] Secondary has no health check
[ ] TTL = 60

[ ] app.demodomain.fun resolves to Mumbai
[ ] http://app.demodomain.fun returns Mumbai application
[ ] http://app.demodomain.fun/health returns OK
```

---

# 71. Failure Test – Exact Order

Run:

```bash
sudo systemctl stop nginx
```

Immediately verify the direct Mumbai ALB path.

Then monitor Route 53.

Wait until:

```text
hc-mumbai = Unhealthy
```

Do not judge the failover based only on the browser.

First check:

```bash
dig @1.1.1.1 +short app.demodomain.fun
```

Expected:

```text
hyd-alb-1968825055.ap-south-2.elb.amazonaws.com.
```

Then:

```bash
curl -i http://app.demodomain.fun
```

Expected:

```text
HYDERABAD SECONDARY SERVER
Account B - ap-south-2
```

Then:

```bash
curl -i http://app.demodomain.fun/health
```

Expected:

```text
200
OK
```

---

# 72. Failover Troubleshooting Order

If DNS still returns Mumbai, do not immediately recreate everything.

Check in this exact order:

### 1. Confirm Mumbai is actually unhealthy

```text
Route 53
→ Health checks
→ hc-mumbai
```

Expected:

```text
Unhealthy
```

### 2. Confirm the Primary record uses hc-mumbai

The Primary record must show:

```text
Health check:
hc-mumbai
```

### 3. Confirm Secondary has no health check

The Secondary record must show:

```text
Health check:
blank
```

### 4. Confirm both records are Failover records

Verify:

```text
Primary = Failover / Primary
Secondary = Failover / Secondary
```

### 5. Confirm no duplicate/conflicting application record

The hosted zone should not contain another ordinary/weighted/latency record for:

```text
app.demodomain.fun
```

### 6. Check the authoritative Route 53 nameserver

Example:

```bash
dig @ns-1673.awsdns-17.co.uk app.demodomain.fun
```

If the authoritative answer is already Hyderabad but `@1.1.1.1` still shows Mumbai, the issue is resolver caching/propagation rather than the Route 53 failover configuration.

### 7. Query multiple resolvers

```bash
dig @1.1.1.1 +short app.demodomain.fun
dig @8.8.8.8 +short app.demodomain.fun
```

### 8. Allow TTL/cache time

With:

```text
TTL = 60
```

a previously cached resolver response can temporarily remain visible.

---

# 73. Final Recovery Test – Exact Order

After Hyderabad is verified:

```bash
sudo systemctl start nginx
sudo systemctl status nginx
curl -i http://localhost/health
```

Then verify the Mumbai ALB:

```bash
curl -i http://mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com/health
```

Wait for:

```text
hc-mumbai = Healthy
```

Then:

```bash
dig @1.1.1.1 +short app.demodomain.fun
```

Expected:

```text
mum-app-primary-1426974919.ap-south-1.elb.amazonaws.com.
```

Finally:

```bash
curl -i http://app.demodomain.fun
curl -i http://app.demodomain.fun/health
```

Expected:

```text
MUMBAI PRIMARY SERVER
Account A - ap-south-1

HTTP 200
OK
```

---

# 74. What This Lab Proves vs What It Does Not Prove

## Proven by the completed test

```text
DNS delegation
+
Route 53 failover routing
+
Route 53 health checking
+
Mumbai ALB availability
+
Hyderabad ALB availability
+
Cross-region application endpoint failover
+
DNS failback
```

## Not proven by this lab

```text
Database replication
Database consistency
RDS cross-region failover
Data loss/recovery point objective
Recovery time objective under a formal DR process
Automatic application redeployment
Infrastructure recreation
HTTPS/TLS
WAF protection
Centralized monitoring
Automated disaster recovery orchestration
```

This distinction is important when presenting the project as a DR solution.
