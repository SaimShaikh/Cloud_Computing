
# Setup VPN to Access Private EC2 Server

# Table of Contents

1. Introduction
2. What is a VPN?
3. Why do we need a VPN?
4. Benefits of VPN
5. VPN Workflow
6. AWS VPN Components
7. Architecture
8. Hands-on Lab
9. Verification
10. Real-world Scenario
11. Best Practices
12. Interview Questions

---

# Introduction

In AWS, production servers are usually deployed inside **Private Subnets**.

These servers:

* Do not have a Public IP
* Cannot be accessed directly from the Internet
* Are protected from external attacks

To securely access these private servers, organizations commonly use a **Virtual Private Network (VPN)**.

---

# What is a VPN?

A **Virtual Private Network (VPN)** creates an **encrypted tunnel** between your local computer or office network and your AWS VPC.

Once connected, your laptop behaves as if it is inside the AWS network.

Example:

```text
Laptop
192.168.1.10

↓

Encrypted VPN Tunnel

↓

AWS VPC

↓

Private EC2
10.0.2.100
```

---

# Why Do We Need a VPN?

Suppose your EC2 server is in a private subnet.

```text
Private EC2

Private IP
10.0.2.100

No Public IP
```

Trying to SSH from your laptop:

```bash
ssh ec2-user@10.0.2.100
```

will fail because the private IP is not reachable over the internet.

A VPN securely bridges this gap.

---

# Benefits of VPN

* Secure encrypted communication
* No Public IP required
* No Bastion Host required
* Reduced attack surface
* Secure administrator access
* Corporate network integration
* Supports multiple private resources

---

# AWS VPN Components

| Component               | Purpose                                    |
| ----------------------- | ------------------------------------------ |
| Customer Gateway        | Represents your office or local VPN device |
| Virtual Private Gateway | AWS side VPN endpoint                      |
| VPN Connection          | Encrypted tunnel between AWS and customer  |
| Route Tables            | Direct traffic through VPN                 |
| Private EC2             | Target server                              |

---

# VPN Workflow

```text
Administrator Laptop
        │
        │
Internet
        │
        ▼
Encrypted VPN Tunnel
        │
        ▼
Virtual Private Gateway
        │
        ▼
AWS VPC
        │
        ▼
Private Subnet
        │
        ▼
Private EC2 Server
```

---

# Architecture

```text
                    Internet
                         │
                         │
              +------------------+
              | Administrator PC |
              +------------------+
                         │
                  VPN Tunnel
               (Encrypted IPSec)
                         │
                         ▼
             +----------------------+
             | Virtual Private      |
             | Gateway (AWS VPN)    |
             +----------------------+
                         │
                    VPC 10.0.0.0/16
                         │
          ------------------------------
          │                            │
          │                            │
Public Subnet                  Private Subnet
          │                            │
          │                    +----------------+
          │                    | Private EC2    |
          │                    |10.0.2.100      |
          │                    +----------------+
```

---

# Hands-on Lab

## Requirement

* AWS Account
* Existing VPC
* Private EC2
* Customer Gateway (office firewall or VPN appliance)

---

## Step 1: Create Virtual Private Gateway

Navigate to:

```
VPC

↓

Virtual Private Gateways

↓

Create Virtual Private Gateway
```

Example:

```
Name

Production-VGW
```

Attach it to your VPC.

### Why?

The Virtual Private Gateway is the AWS endpoint for the VPN connection.

---

## Step 2: Create Customer Gateway

Navigate to:

```
VPC

↓

Customer Gateways

↓

Create Customer Gateway
```

Provide:

* Public IP of your office firewall/router
* BGP ASN (or static routing)

### Why?

AWS needs to know where the VPN tunnel terminates on your side.

---

## Step 3: Create VPN Connection

Navigate to:

```
VPC

↓

Site-to-Site VPN

↓

Create VPN Connection
```

Select:

* Virtual Private Gateway
* Customer Gateway

AWS generates two VPN tunnels for high availability.

---

## Step 4: Download Configuration

Choose your VPN device vendor:

* Cisco
* Fortinet
* Palo Alto
* pfSense
* Sophos
* Generic

Download the configuration file.

Apply it to your firewall.

---

## Step 5: Update Route Table

Private subnet route table:

```
Destination

192.168.1.0/24

Target

Virtual Private Gateway
```

This tells AWS how to send traffic back to your office network.

---

## Step 6: Update Security Group

Allow SSH only from your office network.

Example:

```
TCP 22

Source

192.168.1.0/24
```

Do **not** allow `0.0.0.0/0` for SSH.

---

## Step 7: Connect VPN

Connect your laptop to the corporate VPN.

Verify:

```
Connected
```

---

## Step 8: SSH to Private EC2

```bash
ssh ec2-user@10.0.2.100
```

You can now securely access the private server.

---

# Verification

On your laptop:

```bash
ping 10.0.2.100
```

```bash
ssh ec2-user@10.0.2.100
```

Inside EC2:

```bash
hostname

ip addr

uptime
```

The connection should work through the encrypted VPN tunnel.

---

# Real-World Scenario

A company hosts all production servers in private subnets.

Employees first connect to the company VPN.

Once connected, they can:

* SSH to Linux servers
* Access internal applications
* Connect to databases
* Access Kubernetes clusters

No server has a public IP address, significantly improving security.

---

# Best Practices

* Use MFA for VPN authentication.
* Restrict VPN access to authorized users.
* Allow SSH only from VPN CIDR ranges.
* Monitor VPN logs.
* Rotate VPN credentials regularly.
* Use two VPN tunnels for high availability.
* Prefer IAM Identity Center or certificate-based authentication where possible.

---

# Alternative Methods to Access Private EC2

| Method                              | Recommended               |
| ----------------------------------- | ------------------------- |
| Site-to-Site VPN                    | ✅ Enterprise              |
| AWS Client VPN                      | ✅ Remote users            |
| AWS Systems Manager Session Manager | ✅ Best for administration |
| Bastion Host                        | ⚠️ Legacy approach        |
| Direct Connect                      | ✅ Large enterprises       |

---

# Interview Questions

## What is a VPN?

A VPN creates a secure encrypted tunnel between a client network and AWS, allowing access to private resources.

---

## Why use a VPN for private EC2?

Because private EC2 instances do not have public IP addresses and are not directly accessible from the internet.

---

## What is a Virtual Private Gateway?

It is the AWS-side endpoint that terminates the VPN connection.

---

## What is a Customer Gateway?

It represents the customer or on-premises VPN device that connects to AWS.

---

## Can VPN replace a Bastion Host?

Yes. Many organizations use VPN or AWS Systems Manager instead of Bastion Hosts for secure administrative access.

---

# Conclusion

A VPN provides secure, encrypted access to private EC2 instances without exposing them to the internet. By connecting your local network or laptop to your AWS VPC through a VPN tunnel, administrators can safely manage production resources while maintaining a strong security posture.
