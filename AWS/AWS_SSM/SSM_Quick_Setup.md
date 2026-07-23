# AWS Systems Manager Quick Setup

## Introduction

AWS Systems Manager (SSM) Quick Setup is a feature that automatically configures AWS Systems Manager on your EC2 instances. Instead of manually configuring services like Session Manager, Patch Manager, Inventory, and CloudWatch Agent, Quick Setup performs these configurations for you using AWS best practices.

It is mainly used to quickly onboard EC2 instances into Systems Manager with minimal manual work.

---

# Why Do We Use Quick Setup?

Normally, setting up Systems Manager requires you to:

- Attach IAM Roles
- Configure Patch Manager
- Configure Inventory
- Configure Session Manager
- Create State Manager Associations
- Install CloudWatch Agent
- Configure Maintenance Windows

Doing this manually for hundreds of EC2 instances takes time.

Quick Setup automates all these tasks.

---

# Prerequisites

Before using Quick Setup, ensure you have:

- AWS Account
- Running EC2 Instance
- Administrator IAM Permission
- SSM Agent installed (Most AWS AMIs already have it)
- IAM Role attached to EC2

Example IAM Policy:

```
AmazonSSMManagedInstanceCore
```

The instance must also have network connectivity to Systems Manager either through:

- Internet Gateway
- NAT Gateway
- VPC Interface Endpoints

Required VPC Endpoints:

- ssm
- ssmmessages
- ec2messages

---

# How Quick Setup Works

```
Administrator
      │
      ▼
AWS Systems Manager
      │
      ▼
Quick Setup
      │
      ▼
Creates Configuration
      │
      ▼
State Manager Associations
      │
      ▼
SSM Agent
      │
      ▼
EC2 Instance
```

The SSM Agent receives the configuration and applies it automatically.

---

# Steps to Configure Quick Setup

## Step 1

Login to AWS Console.

Go to:

```
AWS Systems Manager
```

---

## Step 2

From the left menu select

```
Quick Setup
```

Click

```
Create
```

---

## Step 3

Choose a Configuration Type.

Common options are:

- Host Management
- Patch Policy
- Resource Scheduler
- Distributor
- Default Host Management Configuration

For most users,

Select

```
Host Management
```

---

## Step 4

Provide a Configuration Name.

Example

```
My-SSM-Setup
```

---

## Step 5

Choose the Target Instances.

You can target:

- All Current Instances
- Future Instances
- Instances with Specific Tags

Example

```
Environment = Production
```

Only instances with this tag will receive the configuration.

---

## Step 6

Select Features

Enable the required features.

Recommended:

✅ Session Manager

✅ Patch Manager

✅ Inventory Collection

(Optional)

CloudWatch Agent

---

## Step 7

Review the configuration.

Click

```
Create
```

AWS now creates the required Systems Manager Associations automatically.

---

# What Happens Behind the Scenes?

After clicking Create,

Quick Setup performs the following tasks automatically.

### 1. Checks IAM Role

Verifies that the EC2 instance has

```
AmazonSSMManagedInstanceCore
```

permission.

---

### 2. Creates Associations

AWS creates State Manager Associations.

These associations instruct the SSM Agent to perform tasks automatically.

---

### 3. Contacts the EC2

The SSM Agent receives the configuration over HTTPS (Port 443).

No SSH connection is required.

---

### 4. Applies Configuration

Depending on the selected options,

the SSM Agent may

- Enable Inventory
- Configure Patch Manager
- Configure Session Manager
- Install CloudWatch Agent

---

### 5. Reports Status

After successful configuration,

the instance appears under

```
Systems Manager

↓

Managed Nodes
```

with

```
Ping Status = Online
```

---

# Verify the Setup

Go to

```
Systems Manager

↓

Managed Nodes
```

Check

- Instance Name
- Instance ID
- Platform
- Ping Status
- Agent Version

Ping Status should be

```
Online
```

---

# Test Session Manager

Go to

```
EC2

↓

Select Instance

↓

Connect

↓

Session Manager

↓

Connect
```

If a terminal opens,

Quick Setup has been configured successfully.

---

# Common Problems

## Instance Not Showing

Possible Causes

- IAM Role Missing
- SSM Agent Stopped
- Port 443 Blocked

---

## Ping Status Offline

Possible Causes

- No Internet
- Missing NAT Gateway
- Missing VPC Endpoints
- DNS Resolution Failed

---

## Session Manager Not Working

Check

- SSM Agent Running
- IAM Role Attached
- Port 443
- VPC Endpoints
- Security Groups

---

# Best Practices

- Use IAM Roles instead of Access Keys.
- Keep the SSM Agent updated.
- Use VPC Endpoints for private subnets.
- Enable Session Logging.
- Use Patch Manager for automatic updates.
- Tag your EC2 instances for easier management.

---

# Advantages of Quick Setup

- Easy to configure
- Saves time
- No manual configuration
- Centralized management
- Automatically creates required associations
- Follows AWS best practices
- Works with multiple EC2 instances

---

# Interview Questions

### What is AWS Systems Manager Quick Setup?

Quick Setup is a feature of AWS Systems Manager that automatically configures managed instances with services like Session Manager, Patch Manager, Inventory, and CloudWatch Agent using AWS best practices.

---

### Does Quick Setup install the SSM Agent?

No.

Most AWS AMIs already include the SSM Agent.

Quick Setup configures Systems Manager features and verifies that the instance can be managed. If the agent is missing, you must install it manually.

---

### What IAM Policy is required?

```
AmazonSSMManagedInstanceCore
```

---

### Which Port is required?

```
TCP 443
```

---

### Does Quick Setup require SSH?

No.

It communicates through the SSM Agent over HTTPS.

---

# Conclusion

AWS Systems Manager Quick Setup is the fastest way to configure Systems Manager on EC2 instances. It automatically creates the necessary configurations for Session Manager, Patch Manager, Inventory, and other SSM features, reducing manual effort and ensuring consistent management across your infrastructure.
