# AWS EC2 Instance Connect Endpoint (EICE) 

## 1. What is EICE?

**EC2 Instance Connect Endpoint (EICE)** is an AWS-managed
**identity-aware TCP proxy** that lets an authorized user create a
private tunnel from their computer to an EC2 instance.

It is mainly useful when an EC2 instance has only a private IP and you
want to connect using **SSH or RDP**, without giving the instance a
public IP or maintaining a Bastion Host.

> **Easy definition:** EICE gives you a controlled network path to a
> private EC2 so you can use SSH/RDP.

------------------------------------------------------------------------

## 2. The problem EICE solves

Example:

``` text
VPC: 10.0.0.0/16

Private Subnet
10.0.2.0/24

EC2
Private IP: 10.0.2.10
Public IP: None
```

Your laptop is outside the VPC:

``` text
Your Laptop
     |
  Internet
     |
     X
     |
Private EC2
10.0.2.10
```

You cannot simply SSH to `10.0.2.10` from the internet because it is a
private address.

------------------------------------------------------------------------

## 3. Traditional solution: Bastion Host

``` text
Laptop
   |
Internet
   |
Bastion Host
   |
   | SSH
   v
Private EC2
```

The Bastion provides the jump point, but you must manage another EC2
instance.

EICE can remove the need for that dedicated Bastion.

------------------------------------------------------------------------

## 4. EICE solution

``` text
Your Laptop
     |
     | Authenticated tunnel
     v
EC2 Instance Connect Endpoint
     |
     | TCP traffic
     v
Private EC2
10.0.2.10
```

The target EC2 can remain:

``` text
Public IP: None
Private IP: 10.0.2.10
```

EICE can be used without requiring the VPC to have direct internet
connectivity through an Internet Gateway.

------------------------------------------------------------------------

## 5. What exactly is the endpoint?

Think of EICE as a **controlled doorway/network entry point into your
VPC**.

AWS creates a network interface for the endpoint in the selected subnet.
Routing and security groups determine which target instances the
endpoint can reach.

The endpoint itself is not your application/server EC2.

------------------------------------------------------------------------

## 6. EICE does NOT directly talk to the Linux kernel

This is an important correction.

EICE does not bypass the operating system.

For SSH, think of the path as:

``` text
Your Laptop
     |
     v
EICE service / private tunnel
     |
     v
EC2 network interface
     |
     v
SSH server (sshd)
     |
     v
Shell
     |
     v
Operating System
     |
     v
Linux Kernel
```

> **EICE solves the network-connectivity problem. It does not directly
> communicate with the kernel.**

------------------------------------------------------------------------

## 7. EICE vs EC2 Instance Connect

These names are easy to confuse.

### EC2 Instance Connect

EC2 Instance Connect can provide temporary SSH public-key access to a
Linux instance.

Conceptually:

``` text
IAM permissions
      |
      v
EC2 Instance Connect
      |
      v
Temporary SSH public key
      |
      v
EC2 / sshd
```

### EC2 Instance Connect Endpoint

EICE is the **network path/proxy** that lets you reach the private
instance.

They can be used together:

``` text
EC2 Instance Connect
       +
EICE
       =
Temporary-key authentication
through a private connectivity path
```

------------------------------------------------------------------------

## 8. Does EICE always require EC2 Instance Connect software?

**No.**

This is an important distinction.

If you use EICE with your **own SSH key and normal SSH authentication**,
the EC2 Instance Connect package is not necessarily required.

If you use the **EC2 Instance Connect temporary-key mechanism**, the
required EC2 Instance Connect software must be available on the Linux
instance.

Many supported AWS AMIs already have it installed.

------------------------------------------------------------------------

## 9. EICE supports SSH and RDP

EICE is not Linux-only.

``` text
Linux EC2
   |
   +--> SSH

Windows EC2
   |
   +--> RDP
```

So the general idea is:

> EICE provides private connectivity to an EC2 instance for management
> protocols such as SSH or RDP.

------------------------------------------------------------------------

## 10. EICE vs SSM

### SSM Session Manager

SSM uses the **SSM Agent** running inside the EC2.

``` text
Your Laptop
     |
     v
AWS Systems Manager
     |
     v
SSM Agent
     |
     v
EC2 Operating System
     |
     v
Shell
```

### EICE

EICE provides a network path to the EC2.

``` text
Your Laptop
     |
     v
EICE
     |
     v
Private network path
     |
     v
EC2
     |
     v
SSH / RDP
```

### Easy memory trick

> **SSM = agent-based management**

> **EICE = network connectivity for SSH/RDP**

------------------------------------------------------------------------

## 11. Why use EICE if SSM exists?

If someone says:

> "I specifically need SSH access to this private server."

EICE is a natural fit.

If someone says:

> "I only need a shell for administration and I don't specifically need
> SSH."

SSM Session Manager is often the cleaner option to evaluate.

------------------------------------------------------------------------

## 12. EICE vs Bastion vs SSM

  -----------------------------------------------------------------------
  Feature           EICE              Bastion           SSM Session
                                                        Manager
  ----------------- ----------------- ----------------- -----------------
  Target EC2 can    Yes               Yes               Yes
  remain private                                        

  Target public IP  No                No                No
  required                                              

  Dedicated Bastion No                Yes               No
  required                                              

  SSH               Yes               Yes               Not required

  RDP               Yes               Yes               Different SSM
                                                        management
                                                        workflows

  SSM Agent         No                No                Yes
  required                                              

  IAM authorization Yes               Depends on design Yes

  Extra EC2 to      No                Yes               No
  manage                                                

  Best fit          Private SSH/RDP   Traditional       Agent-based
                                      jump-host design  administration
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 13. Decision tree

``` text
Need to reach a private EC2?
            |
           YES
            |
            v
Do you specifically need SSH/RDP?
        /                   YES              NO
       |                |
       v                v
     EICE               SSM
```

### Example

"I need to SSH into my private Linux server."

→ **EICE**

"I need a secure shell for administration and don't care about SSH."

→ **SSM Session Manager**

------------------------------------------------------------------------

## 14. Example architecture

``` text
                    AWS VPC
        +----------------------------+
        |                            |
        |       Private Subnet       |
        |                            |
        |   +------------------+     |
        |   | EICE Endpoint    |     |
        |   | ENI              |     |
        |   +--------+---------+     |
        |            |               |
        |            | TCP           |
        |            v               |
        |   +------------------+     |
        |   | Private EC2      |     |
        |   | 10.0.2.10        |     |
        |   +------------------+     |
        |                            |
        +----------------------------+
                     ^
                     |
              Authenticated
                 tunnel
                     |
                  Laptop
```

------------------------------------------------------------------------

## 15. Networking requirements

Creating EICE does not automatically make every EC2 reachable.

Consider:

-   VPC
-   endpoint subnet
-   route tables
-   target EC2 private IP
-   target EC2 security group
-   endpoint security group
-   IAM permissions
-   SSH/RDP configuration
-   compatible IP address type

The endpoint can reach instances in other subnets of the same VPC when
routing permits it.

------------------------------------------------------------------------

## 16. Security Groups

There are two important sides:

``` text
EICE Security Group
        |
        | OUTBOUND
        v
Target EC2 Security Group
        |
        | INBOUND
        v
SSH 22 / RDP 3389
```

For example, for SSH:

### EICE security group

``` text
Outbound
TCP 22
Destination: target EC2 security group
```

### Target EC2 security group

``` text
Inbound
TCP 22
Source: EICE security group
```

Using a security-group reference is a clean design for many setups.

AWS also documents different source-address behavior when client IP
preservation is enabled or disabled.

Do not blindly open:

``` text
TCP 22
0.0.0.0/0
```

------------------------------------------------------------------------

## 17. IAM permissions

EICE is also an authorization feature.

The user needs the appropriate IAM permissions to use the endpoint.

If using EC2 Instance Connect to push a temporary SSH public key,
permissions such as:

``` text
ec2-instance-connect:SendSSHPublicKey
```

are relevant.

The `ec2:osuser` condition can restrict which operating-system user can
receive the key.

Follow least privilege.

------------------------------------------------------------------------

## 18. CloudTrail

AWS states that **successful and unsuccessful EICE connection attempts
are logged in CloudTrail**.

This is useful for auditing:

``` text
Who?
When?
Which endpoint?
Which target?
Successful or unsuccessful?
```

------------------------------------------------------------------------

## 19. Creating EICE from the console

High-level process:

### Step 1

Go to:

``` text
EC2
  -> Network & Security
  -> Endpoints
```

### Step 2

Choose:

``` text
Create endpoint
```

Select:

``` text
EC2 Instance Connect Endpoint
```

### Step 3

Select the VPC.

### Step 4

Select the subnet where the endpoint network interface will be created.

### Step 5

Choose the IP address type:

``` text
IPv4
Dualstack
IPv6
```

The endpoint's IP type must be compatible with the target instance.

### Step 6

Configure the endpoint security group.

### Step 7

Create the endpoint.

Wait until it becomes:

``` text
Available
```

Then it can be used.

------------------------------------------------------------------------

## 20. Connecting through the AWS Console

For a supported target:

``` text
EC2
  |
Instances
  |
Select instance
  |
Connect
  |
EC2 Instance Connect Endpoint
```

Select the appropriate endpoint and connection settings.

For Linux, the connection can use SSH.

For Windows, the supported EICE workflow can use RDP.

------------------------------------------------------------------------

## 21. Connecting with AWS CLI

AWS CLI v2 supports an EC2 Instance Connect SSH command that can
explicitly use EICE.

Example:

``` bash
aws ec2-instance-connect ssh   --instance-id i-1234567890example   --connection-type eice
```

You can also specify your own private key:

``` bash
aws ec2-instance-connect ssh   --instance-id i-1234567890example   --private-key-file /path/to/key.pem
```

The important option is:

``` text
--connection-type eice
```

This tells the command to use the EC2 Instance Connect Endpoint for the
private connection.

------------------------------------------------------------------------

## 22. EICE does not make the EC2 public

Before:

``` text
Private IP: 10.0.2.10
Public IP: None
```

After creating EICE:

``` text
Private IP: 10.0.2.10
Public IP: None
```

The EC2 remains private.

------------------------------------------------------------------------

## 23. Does the VPC need an Internet Gateway?

Not necessarily for EICE connectivity.

AWS states that EICE can allow connections from the internet without
requiring the VPC to have direct internet connectivity through an
Internet Gateway.

But:

``` text
EICE connectivity
        !=
EC2 internet access
```

EICE does not automatically give your private EC2 general internet
access.

------------------------------------------------------------------------

## 24. EICE is for management traffic

EICE is intended for **management traffic**, not high-volume data
transfers.

Good examples:

``` text
SSH
RDP
Troubleshooting
Configuration checks
Service status
Logs
```

Bad fit:

``` text
Large file transfers
Bulk data movement
Application data pipelines
```

AWS states that high-volume data transfers through EICE are throttled.

------------------------------------------------------------------------

## 25. Important quotas

Current AWS documentation lists:

-   Maximum **5 EICE endpoints per AWS account per Region**
-   Maximum **1 EICE endpoint per VPC**
-   Maximum **1 EICE endpoint per subnet**
-   Maximum **20 concurrent connections per endpoint**
-   Maximum established TCP connection duration: **3,600 seconds / 1
    hour**

Check current AWS quotas before designing a large environment.

------------------------------------------------------------------------

## 26. Availability and subnet design

A key design point:

> You can create only **one EICE endpoint per VPC**.

That means you should not design a normal architecture with multiple
EICE endpoints in multiple AZs inside the same VPC.

One endpoint can reach instances in other subnets of the same VPC when
routing allows the traffic.

Plan the endpoint subnet, routes, and security groups carefully.

------------------------------------------------------------------------

## 27. Cost

AWS currently documents:

> **There is no additional charge for using EC2 Instance Connect
> Endpoints.**

However, applicable cross-AZ data-transfer charges can apply when using
an endpoint to connect to an instance in a different Availability Zone.

Always verify current AWS pricing before production deployment.

------------------------------------------------------------------------

## 28. Production scenario

Suppose:

``` text
                Internet
                    |
              Load Balancer
                    |
                    v
              Private Subnet
          +---------------------+
          |                     |
          | App Server 1        |
          | App Server 2        |
          | App Server 3        |
          |                     |
          +---------------------+
```

None of the application servers have public IPs.

An engineer needs to troubleshoot App Server 2.

### Old approach

``` text
Engineer
   |
   v
Bastion
   |
   v
App Server 2
```

### EICE approach

``` text
Engineer
   |
   v
EICE
   |
   v
App Server 2
```

### SSM approach

``` text
Engineer
   |
   v
Systems Manager
   |
   v
SSM Agent
   |
   v
App Server 2
```

If SSH is specifically required:

``` text
EICE
```

If normal administration is enough:

``` text
SSM
```

------------------------------------------------------------------------

## 29. Windows scenario

Private Windows EC2:

``` text
Private IP: 10.0.3.20
Public IP: None
```

Need RDP:

``` text
Laptop
  |
  v
EICE
  |
  v
RDP
  |
  v
Windows EC2
```

This is another reason not to think of EICE as "only SSH."

------------------------------------------------------------------------

## 30. When EICE is the best fit

EICE is a strong fit when:

``` text
✓ Instance is private
✓ No public IP is desired
✓ SSH/RDP is specifically required
✓ You don't want a Bastion Host
✓ IAM-controlled access is desired
✓ Traffic is management traffic
```

------------------------------------------------------------------------

## 31. When SSM is a better fit

Evaluate SSM first when:

``` text
✓ You don't specifically need SSH/RDP
✓ You want centralized session management
✓ You want IAM-based access
✓ You want to avoid SSH-key workflows
✓ You want Session Manager logging/auditing
✓ SSM prerequisites are already available
```

------------------------------------------------------------------------

## 32. When a Bastion can still make sense

EICE does not automatically make every Bastion architecture wrong.

A Bastion may still exist because of:

-   legacy SSH workflows
-   third-party tools
-   enterprise standards
-   existing architecture
-   specific network/security requirements

But if the only reason for the Bastion is:

> "We need a way to SSH into private EC2."

then evaluate **EICE and SSM** before adding another EC2 server.

------------------------------------------------------------------------

## 33. Common mistakes

### Mistake 1 --- "EICE gives the EC2 a public IP"

Wrong.

``` text
EICE
  ↓
Private EC2
```

The EC2 can remain private.

### Mistake 2 --- "EICE directly talks to the kernel"

Wrong.

``` text
EICE
 ↓
Network
 ↓
SSH/RDP
 ↓
Operating System
 ↓
Kernel
```

### Mistake 3 --- "EICE is the same as SSM"

Wrong.

``` text
EICE → private network connectivity

SSM → agent-based management
```

### Mistake 4 --- "EICE bypasses security groups"

Wrong.

Security groups still control traffic.

### Mistake 5 --- "I should open SSH to 0.0.0.0/0"

Avoid this for private-management designs.

Use controlled sources and IAM authorization.

------------------------------------------------------------------------

## 34. Interview questions

### Q1. What is EICE?

> EC2 Instance Connect Endpoint is an AWS-managed identity-aware TCP
> proxy that creates a private tunnel to an EC2 instance, allowing
> authorized users to connect to private instances using SSH or RDP
> without requiring a public IP or Bastion Host.

### Q2. Does EICE make the EC2 public?

> No. The EC2 can remain private with only a private IP.

### Q3. EICE vs SSM?

> EICE provides network connectivity for SSH/RDP, while SSM Session
> Manager uses the SSM Agent for agent-based instance management.

### Q4. Do I need a Bastion with EICE?

> No. One of EICE's main benefits is that it can remove the need for a
> dedicated Bastion Host.

### Q5. Does EICE require the SSM Agent?

> No. EICE and SSM are separate mechanisms.

### Q6. Does EICE always require EC2 Instance Connect software?

> No. It depends on the authentication method. Normal SSH with your own
> key does not necessarily require the EC2 Instance Connect package; the
> temporary-key EC2 Instance Connect method does.

------------------------------------------------------------------------

