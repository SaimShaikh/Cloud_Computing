# AWS WAF + Application Load Balancer + EC2 (Nginx) Security Project

## Project Overview

This project demonstrates how to secure a web application hosted on an AWS EC2 instance using AWS WAF (Web Application Firewall) and an Application Load Balancer (ALB).

The application is hosted on an EC2 instance running Nginx, while AWS WAF is used to inspect and filter incoming HTTP traffic before it reaches the backend server.

This architecture represents a real-world production-style deployment used by many organizations to improve security, scalability, and availability.

---

# Architecture

```text
Users
   ↓
AWS WAF
   ↓
Application Load Balancer (ALB)
   ↓
Target Group
   ↓
EC2 Instance (Nginx)
```

---

# Project Objectives

The main goals of this project are:

* Deploy a web application using Nginx on EC2
* Configure an Application Load Balancer
* Register EC2 instance in a Target Group
* Configure Health Checks
* Implement AWS WAF protection
* Create custom WAF rules
* Block specific IP addresses
* Configure CAPTCHA challenge rules
* Understand Layer 7 traffic filtering
* Learn real-world AWS security architecture

---

# AWS Services Used

| Service                   | Purpose                       |
| ------------------------- | ----------------------------- |
| Amazon EC2                | Host the Nginx web server     |
| Nginx                     | Serve web application         |
| Application Load Balancer | Route incoming traffic        |
| Target Group              | Manage backend EC2 targets    |
| AWS WAF                   | Filter malicious HTTP traffic |
| CloudWatch                | Monitor WAF metrics           |
| Security Groups           | Control network-level access  |

---

# Understanding the Architecture

## 1. User Request

A client sends an HTTP request to the application.

```text
User → Application URL
```

---

## 2. AWS WAF Inspection

AWS WAF inspects incoming traffic before forwarding it.

WAF can:

* Block IP addresses
* Apply CAPTCHA
* Detect malicious requests
* Filter traffic based on custom rules
* Protect against Layer 7 attacks

---

## 3. Application Load Balancer

The ALB receives requests allowed by WAF.

Responsibilities:

* Route traffic
* Perform health checks
* Distribute requests
* Improve availability
* Support Layer 7 routing

---

## 4. Target Group

The Target Group contains backend EC2 instances.

ALB forwards traffic only to healthy targets.

---

## 5. EC2 + Nginx

The EC2 instance hosts the Nginx web server that serves the application.

---

# Step-by-Step Implementation

# Step 1 — Launch EC2 Instance

Navigate to:

```text
AWS Console → EC2 → Launch Instance
```

## Configuration

### Name

```text
nginx-server
```

### AMI

```text
Ubuntu Server 24.04 LTS
```

### Instance Type

```text
t2.micro
```

### Key Pair

Create and download a key pair:

```text
nginx-key.pem
```

### Network

Use:

```text
Default VPC
```

### Security Group Configuration

Allow the following inbound rules:

| Type | Port | Source   |
| ---- | ---- | -------- |
| SSH  | 22   | My IP    |
| HTTP | 80   | Anywhere |

Launch the instance.

---

# Step 2 — Connect to EC2

Use SSH to connect:

```bash
chmod 400 nginx-key.pem
```

```bash
ssh -i nginx-key.pem ubuntu@YOUR_PUBLIC_IP
```

---

# Step 3 — Install Nginx

Update packages:

```bash
sudo apt update -y
```

Install Nginx:

```bash
sudo apt install nginx -y
```

Start Nginx:

```bash
sudo systemctl start nginx
```

Enable Nginx at boot:

```bash
sudo systemctl enable nginx
```

Check status:

```bash
sudo systemctl status nginx
```

Expected Output:

```text
active (running)
```

---

# Step 4 — Verify Nginx

Open browser:

```text
http://EC2_PUBLIC_IP
```

Expected:

```text
Welcome to nginx!
```

---

# Step 5 — Create Custom Web Page

Edit Nginx default page:

```bash
sudo nano /var/www/html/index.nginx-debian.html
```

Add:

```html
<!DOCTYPE html>
<html>
<head>
<title>AWS WAF Demo</title>
</head>
<body>
<h1>AWS WAF + ALB + NGINX</h1>
<p>Application Protected Using AWS WAF</p>
</body>
</html>
```

Save and refresh browser.

---

# Step 6 — Create Target Group

Navigate to:

```text
EC2 → Target Groups → Create Target Group
```

## Configuration

### Target Type

```text
Instances
```

### Name

```text
nginx-target-group
```

### Protocol

```text
HTTP
```

### Port

```text
80
```

### VPC

```text
Default VPC
```

### Health Check Path

```text
/
```

### Register Targets

Select the EC2 instance and register it.

Create the Target Group.

---

# Understanding Health Checks

The ALB continuously checks if the EC2 instance is healthy.

Example request:

```http
GET /
```

If the server responds with:

```text
200 OK
```

then the target is considered healthy.

If health checks fail:

* ALB marks target unhealthy
* Traffic is stopped to that instance

---

# Step 7 — Create Application Load Balancer

Navigate to:

```text
EC2 → Load Balancers → Create Load Balancer
```

Choose:

```text
Application Load Balancer
```

---

# ALB Configuration

## Name

```text
nginx-alb
```

## Scheme

```text
Internet-facing
```

## IP Address Type

```text
IPv4
```

## Network Mapping

Choose:

```text
Default VPC
```

Select at least two Availability Zones.

Example:

* ap-south-1a
* ap-south-1b

---

# Security Group for ALB

Allow:

| Type | Port | Source   |
| ---- | ---- | -------- |
| HTTP | 80   | Anywhere |

---

# Listener Configuration

Protocol:

```text
HTTP
```

Port:

```text
80
```

Forward requests to:

```text
nginx-target-group
```

Create the ALB.

---

# Step 8 — Verify Target Health

Navigate to:

```text
EC2 → Target Groups → Targets
```

Expected:

```text
Healthy
```

---

# Step 9 — Test ALB

Copy the ALB DNS name.

Example:

```text
http://nginx-alb-123.ap-south-1.elb.amazonaws.com
```

Open in browser.

The Nginx page should load successfully.

---

# Step 10 — Create AWS WAF Web ACL

Navigate to:

```text
AWS WAF & Shield → Web ACLs → Create Web ACL
```

---

# WAF Configuration

## Name

```text
waf_ng
```

## Scope

```text
Regional
```

## Region

```text
ap-south-1
```

## Associate AWS Resource

Select:

```text
nginx-alb
```

---

# Understanding WAF Scope

## Regional Scope

Used with:

* ALB
* API Gateway
* App Runner

## CloudFront Scope

Used only with:

* CloudFront distributions

---

# Step 11 — Create Custom IP Blocking Rule

Navigate to:

```text
AWS WAF → IP Sets → Create IP Set
```

---

# IP Set Configuration

## Name

```text
blocked-ips
```

## Region

```text
ap-south-1
```

## IP Version

```text
IPv4
```

## Add IP Address

Example:

```text
49.xx.xx.xx/32
```

### Why /32?

```text
/32 = Single IP Address
```

Create the IP Set.

---

# Step 12 — Create WAF Rule

Navigate to:

```text
AWS WAF → Web ACLs → waf_ng
```

Click:

```text
Add Rules → Add my own rules and rule groups
```

---

# Rule Configuration

## Rule Type

```text
Regular Rule
```

## Rule Name

```text
Ip-block
```

## Statement

Choose:

```text
IP Set
```

Select:

```text
blocked-ips
```

## Action

```text
Block
```

Save the rule.

---

# Step 13 — Test IP Blocking

Open the ALB DNS URL.

Expected:

```text
403 Forbidden
```
<img width="1671" height="388" alt="Screenshot 2026-05-16 at 12 50 18 PM" src="https://github.com/user-attachments/assets/a0775389-f43f-4814-82b3-b1170f500156" />

This confirms:

* WAF blocked the request
* Traffic never reached EC2
* Layer 7 filtering works successfully

---

# Step 14 — Configure CAPTCHA Rule

Instead of blocking users directly, AWS WAF can challenge users with CAPTCHA.

Navigate to:

```text
AWS WAF → Web ACLs → Add Rule
```

---

# CAPTCHA Rule Configuration

## Rule Type

```text
Regular Rule
```

## Match Condition

Example:

```text
URI Path contains /
```

## Action

```text
CAPTCHA
```

Save the rule.

---

# CAPTCHA Flow

```text
User Request
     ↓
AWS WAF
     ↓
CAPTCHA Challenge
     ↓
User Solves CAPTCHA
     ↓
Access Allowed
     ↓
ALB
     ↓
Nginx
```

## Output 

<img width="1670" height="586" alt="Screenshot 2026-05-16 at 12 51 24 PM" src="https://github.com/user-attachments/assets/eb26c991-c444-4b7f-b507-1a600b4821e8" />

<img width="1674" height="352" alt="Screenshot 2026-05-16 at 12 51 59 PM" src="https://github.com/user-attachments/assets/51bf8583-dac9-4013-b914-6258af48194b" />

---

# Difference Between BLOCK and CAPTCHA

| Action  | Result                      |
| ------- | --------------------------- |
| BLOCK   | Immediate 403 Forbidden     |
| CAPTCHA | User verification challenge |

---

# Step 15 — Monitor WAF Metrics

Navigate to:

```text
CloudWatch → Metrics → WAF
```

<img width="1364" height="554" alt="Screenshot 2026-05-16 at 1 01 11 PM" src="https://github.com/user-attachments/assets/2baf1db0-aac2-4503-a089-78b2854f7245" />

Monitor:

* Allowed requests
* Blocked requests
* CAPTCHA requests
* Rule matches

---

# Internal Working of the Architecture

## Normal Request Flow

```text
User
 ↓
AWS WAF
 ↓
ALB
 ↓
Target Group
 ↓
EC2 (Nginx)
```

---

## Blocked Request Flow

```text
User
 ↓
AWS WAF
 ↓
IP Match Detected
 ↓
403 Forbidden
```

---

# Security Concepts Learned

This project demonstrates:

* Layer 7 security filtering
* Web traffic inspection
* Reverse proxy architecture
* Load balancing concepts
* Health checks
* Custom WAF rule creation
* CAPTCHA implementation
* IP reputation filtering
* High availability concepts

---

# Real-World Production Improvements

Production environments usually include:

* HTTPS
* ACM SSL certificates
* Route53
* Auto Scaling Group
* CloudFront CDN
* Private subnets
* Logging and monitoring
* SIEM integration
* Multi-AZ deployment

---

# Common Troubleshooting

# Issue: Target Group Unhealthy

Possible causes:

* Nginx not running
* Security Group blocking port 80
* Wrong health check path
* EC2 instance stopped

Verify:

```bash
sudo systemctl status nginx
```

---

# Issue: ALB Not Accessible

Check:

* ALB Security Group
* Listener configuration
* Public subnets selected

---

# Issue: WAF Not Blocking

Check:

* Rule attached correctly
* Web ACL associated with ALB
* Correct IP in IP set
* Wait for propagation

---

# Key Learning Outcomes

After completing this project, you will understand:

* AWS WAF architecture
* ALB traffic flow
* Target Group functionality
* Health checks
* Layer 7 filtering
* Security best practices
* Real-world AWS deployment patterns
* Custom WAF rule creation

---

# Conclusion

This project demonstrates how to build a secure, scalable, and production-style web application architecture on AWS using:

* EC2
* Nginx
* Application Load Balancer
* AWS WAF

The implementation provides hands-on experience with:

* web security
* traffic filtering
* load balancing
* custom firewall rules
* CAPTCHA validation
* AWS networking concepts

This architecture is widely used in real-world cloud and DevOps environments to protect applications from malicious traffic while maintaining scalability and availability.

---

# Cleanup

To avoid unnecessary AWS charges, delete the following resources after testing:

1. AWS WAF Web ACL
2. ALB
3. Target Group
4. EC2 Instance
5. Security Groups (optional)

---


