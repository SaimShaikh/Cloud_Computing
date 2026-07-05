# AWS Systems Manager (SSM) Agent Installation & Session Manager — Login to Private EC2 Instance (No SSH, No Bastion, No Public IP)

**Author:** Saime Shaikh
**Topic:** SSM Agent Installation, Verification & Secure Shell Access to a Private Instance via Session Manager
**Level:** Hands-on Lab (Console-First)

---

## Table of Contents

1. Concept Overview
2. Why This Matters (The Problem With SSH + Bastion Hosts)
3. Architecture Diagram
4. Prerequisites
5. Lab Setup — Building the Environment
6. Step 1: Create IAM Role for SSM
7. Step 2: Launch the Private EC2 Instance
8. Step 3: Set Up VPC Interface Endpoints (Required for Private Subnet — No NAT/IGW)
9. Step 4: Verify SSM Agent Is Installed & Running
10. Step 5: Connect to the Private Instance Using Session Manager
11. Step 6: Verify the Session (Prove It's Actually Private)
12. CLI Commands Reference (Equivalent Actions)
13. Terraform Code (Infrastructure as Code)
14. Edge Cases & Failure Scenarios
15. Troubleshooting Checklist
16. Best Practices
17. Interview Questions & Answers
18. Cheat Sheet
19. Mastery Checklist

---

## 1. Concept Overview

AWS Systems Manager **Session Manager** lets you get an interactive shell (or PowerShell) session on an EC2 instance **without**:

- Opening inbound port 22 (SSH) or 3389 (RDP)
- Assigning a public IP address
- Managing SSH key pairs
- Using a bastion/jump host
- Any inbound security group rule at all

Instead, the **SSM Agent** running on the instance makes an **outbound** connection to the AWS Systems Manager service. When you start a session from the console (or CLI), Systems Manager brokers a secure, encrypted, bidirectional channel between your terminal and the instance's shell — entirely over that outbound connection.

**Core components involved:**

| Component | Role |
|---|---|
| **SSM Agent** | Software running on the EC2 instance that talks to the SSM service |
| **IAM Role (Instance Profile)** | Grants the EC2 instance permission to talk to SSM |
| **AmazonSSMManagedInstanceCore** | AWS managed policy that provides the minimum permissions for SSM Agent to function |
| **VPC Interface Endpoints** | Required when the instance has no internet route (no NAT Gateway/IGW) — lets the agent reach SSM privately via AWS PrivateLink |
| **Session Manager** | The Systems Manager feature that brokers the actual shell session |
| **CloudTrail (optional)** | Logs every session start/end — who connected, when |

---

## 2. Why This Matters (The Problem With SSH + Bastion Hosts)

**Traditional private instance access:**

```
Your Laptop → Bastion Host (public IP, SSH open) → Private Instance (SSH)
```

Problems with this model:

- Bastion host is a **permanent attack surface** — it has a public IP and an open SSH port 24/7.
- You must **manage SSH key pairs**, rotate them, and worry about leaked keys.
- You need **security group chaining** (bastion SG → private instance SG).
- No **native session logging** — you'd need to bolt on your own auditing (script session recording, etc.).
- **Compliance headaches** — proving "who accessed what, when" is manual work.

**With SSM Session Manager:**

```
Your Laptop → AWS Systems Manager Service → Private Instance (SSM Agent, outbound only)
```

- **Zero inbound ports.** No SSH, no RDP, no bastion.
- **No SSH keys** to manage or rotate.
- **IAM controls access** — the same IAM policies you already use for AWS.
- **Every session is logged** in CloudTrail; session content can be logged to S3/CloudWatch Logs.
- Works even if the instance is in a **fully private subnet with zero internet access**, as long as VPC Interface Endpoints exist.

---

## 3. Architecture Diagram

### Scenario A: Private subnet WITH NAT Gateway (agent reaches SSM via NAT → IGW)

```
                                   ┌─────────────────────────────────────────┐
                                   │                  AWS Cloud               │
                                   │                                           │
  ┌────────────┐                  │   ┌───────────────────────────────────┐   │
  │  IAM User   │  Console/CLI     │   │           VPC (10.0.0.0/16)        │   │
  │ (You)       │─────────────────┼──▶│                                     │   │
  └────────────┘                  │   │  ┌─────────────────────────────┐    │   │
                                   │   │  │      Public Subnet          │    │   │
                                   │   │  │   ┌───────────────────┐     │    │   │
                                   │   │  │   │   NAT Gateway      │     │    │   │
                                   │   │  │   └─────────┬─────────┘     │    │   │
                                   │   │  └─────────────┼───────────────┘    │   │
                                   │   │                │                    │   │
                                   │   │  ┌─────────────┼───────────────┐    │   │
                                   │   │  │   Private Subnet            │    │   │
                                   │   │  │   ┌─────────▼─────────┐     │    │   │
                                   │   │  │   │  EC2 Instance      │     │    │   │
                                   │   │  │   │  - No Public IP    │     │    │   │
                                   │   │  │   │  - SSM Agent (out) │     │    │   │
                                   │   │  │   │  - IAM Role attach │     │    │   │
                                   │   │  │   └────────────────────┘     │    │   │
                                   │   │  └───────────────────────────────┘    │   │
                                   │   │                                     │   │
                                   │   └───────────────┬─────────────────────┘   │
                                   │                   │ (via Internet Gateway)  │
                                   │           ┌────────▼────────┐               │
                                   │           │  AWS Systems     │               │
                                   │           │  Manager Service │               │
                                   │           └──────────────────┘               │
                                   └─────────────────────────────────────────┘
```

### Scenario B: FULLY PRIVATE subnet — NO NAT, NO IGW (uses VPC Interface Endpoints) — **this is the scenario this lab builds**

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                  AWS Cloud                                │
│                                                                            │
│   ┌────────────┐         ┌────────────────────────────────────────────┐  │
│   │  IAM User   │  HTTPS  │              VPC (10.0.0.0/16)              │  │
│   │  (Console)  │────────▶│                                              │  │
│   └────────────┘         │   ┌──────────────────────────────────────┐   │  │
│                           │   │        Private Subnet (10.0.1.0/24)  │   │  │
│                           │   │                                       │   │  │
│                           │   │   ┌─────────────────────────────┐    │   │  │
│                           │   │   │   EC2 Instance               │    │   │  │
│                           │   │   │   - No Public IP              │    │   │  │
│                           │   │   │   - No NAT/IGW route          │    │   │  │
│                           │   │   │   - SSM Agent (pre-installed) │    │   │  │
│                           │   │   │   - IAM Instance Profile      │    │   │  │
│                           │   │   └──────────────┬────────────────┘    │   │  │
│                           │   │                  │ ENI (private IP)    │   │  │
│                           │   │                  ▼                     │   │  │
│                           │   │   ┌─────────────────────────────┐    │   │  │
│                           │   │   │   VPC Interface Endpoints     │    │   │  │
│                           │   │   │   (PrivateLink, in same VPC)  │    │   │  │
│                           │   │   │                                │    │   │  │
│                           │   │   │  • com.amazonaws.<region>.ssm  │    │   │  │
│                           │   │   │  • com.amazonaws.<region>.ec2messages │  │
│                           │   │   │  • com.amazonaws.<region>.ssmmessages │  │
│                           │   │   └──────────────┬────────────────┘    │   │  │
│                           │   └──────────────────┼─────────────────────┘   │  │
│                           │                      │                          │  │
│                           │                      ▼                          │  │
│                           │            AWS Systems Manager Service          │  │
│                           │            (reached via PrivateLink,            │  │
│                           │             traffic never leaves AWS network)   │  │
│                           └──────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Prerequisites

- AWS account with console access
- IAM permissions to create: IAM roles, EC2 instances, VPCs/Subnets, Security Groups, VPC Endpoints
- A VPC with at least one **private subnet** (this lab builds one from scratch — you can also reuse an existing VPC)
- Region used in this lab: **ap-south-1 (Mumbai)** — substitute your own region, but keep it consistent everywhere
- No SSH key pair needed
- No AWS CLI required for the lab steps (a CLI reference section is included separately at the end for those who want it)

---

## ⚠️ READ THIS BEFORE YOU START — Which Path Are You Doing?

This lab is written for **Scenario B: a fully private subnet with NO NAT Gateway and NO Internet Gateway**. That is the harder, more realistic enterprise pattern, and it is the whole reason **Step 3 (VPC Interface Endpoints) is mandatory, not optional, in this guide.**

Confirm which path applies to you **before you launch the EC2 instance**, because it changes whether Step 3 is required:

| Your Subnet Has... | Path | Do You Need Step 3 (VPC Endpoints)? |
|---|---|---|
| A route to an Internet Gateway directly (public subnet) | Scenario A | ❌ No — skip Step 3 entirely, go straight to Step 4 |
| A route to a NAT Gateway (private subnet, but NAT exists) | Scenario A | ❌ No — skip Step 3 entirely, go straight to Step 4 |
| **No NAT Gateway and no IGW route at all** (fully isolated) | **Scenario B (this guide)** | ✅ **Yes — mandatory.** Without it, the SSM Agent has no path to the SSM service, the instance will never show "Online" in Fleet Manager, and Session Manager will never connect. |

**This guide builds Scenario B from scratch**, including creating a subnet with zero NAT/IGW route on purpose, specifically so you get hands-on practice with VPC Interface Endpoints — this is the part most tutorials skip and the part interviewers actually ask about.

**Sequence matters — this is why the steps are ordered the way they are:** Step 1 (IAM Role) → Step 2 (Launch Instance) → Step 3 (VPC Endpoints) → Step 4 (Verify Agent/Fleet Manager) → Step 5 (Connect). Endpoints are created *before* you ever check Fleet Manager, so you never see a confusing "0 managed nodes" screen for something that isn't actually broken yet.

If you'd rather do the simpler version first, use a subnet that already has a NAT Gateway, skip Step 3 entirely, and go straight from Step 2 to Step 4.

---

## 5. Lab Setup — Building the Environment

We will build this from scratch so the "fully private, no internet route" scenario is proven end-to-end:

1. A VPC
2. A private subnet (no route to IGW/NAT)
3. A Security Group for the EC2 instance (no inbound rules needed at all)
4. A Security Group for the VPC endpoints (allows HTTPS inbound from the instance SG)
5. An IAM Role for SSM
6. An EC2 instance in the private subnet
7. Three VPC Interface Endpoints (ssm, ssmmessages, ec2messages)
8. A Session Manager connection

If you already have a VPC/private subnet you want to reuse, skip to **Step 1** and just make sure your chosen subnet has no route to a NAT Gateway or Internet Gateway (to genuinely test the private scenario), or skip endpoint creation entirely if your subnet already routes to a NAT Gateway.

### 5.1 Create the VPC

1. Go to **VPC console** → **Your VPCs** → **Create VPC**
2. Choose **VPC only**
3. **Name tag:** `ssm-lab-vpc`
4. **IPv4 CIDR block:** `10.0.0.0/16`
5. Leave IPv6 as **No IPv6 CIDR block**
6. Tenancy: **Default**
7. Click **Create VPC**

### 5.2 Create the Private Subnet

1. In VPC console → **Subnets** → **Create subnet**
2. **VPC ID:** select `ssm-lab-vpc`
3. **Subnet name:** `ssm-private-subnet`
4. **Availability Zone:** pick any, e.g. `ap-south-1a`
5. **IPv4 CIDR block:** `10.0.1.0/24`
6. Click **Create subnet**
7. Confirm this subnet's **route table** only has the default `local` route (no `0.0.0.0/0` entry pointing to an IGW or NAT). This is what makes it "private."

---

## 6. Step 1: Create IAM Role for SSM

The EC2 instance needs an IAM role so the SSM Agent on it can authenticate to the Systems Manager service.

1. Go to **IAM console** → **Roles** → **Create role**
2. **Trusted entity type:** AWS service
3. **Use case:** select **EC2** → click **Next**
4. In the permissions policy search box, type: `AmazonSSMManagedInstanceCore`
5. Check the box next to **AmazonSSMManagedInstanceCore**
6. Click **Next**
7. **Role name:** `EC2-SSM-Role`
8. Review and click **Create role**

**What this policy allows (high level):**

- `ssm:UpdateInstanceInformation`
- `ssm:ListAssociations` / `ssm:PutInventory` etc.
- `ssmmessages:CreateControlChannel`, `ssmmessages:CreateDataChannel`, `ssmmessages:OpenControlChannel`, `ssmmessages:OpenDataChannel`
- `ec2messages:GetMessages`, `ec2messages:SendReply`, etc.

These are exactly the API calls the SSM Agent makes internally — nothing more.

---

## 7. Step 2: Launch the Private EC2 Instance

1. Go to **EC2 console** → **Instances** → **Launch instances**
2. **Name:** `ssm-private-instance`
3. **AMI:** Amazon Linux 2023 (SSM Agent comes **pre-installed** on this AMI — ideal for this lab)
4. **Instance type:** `t2.micro` or `t3.micro` (Free Tier eligible)
5. **Key pair:** select **Proceed without a key pair** (not needed — this is the whole point of SSM!)
6. **Network settings** → click **Edit**:
   - **VPC:** `ssm-lab-vpc`
   - **Subnet:** `ssm-private-subnet`
   - **Auto-assign public IP:** **Disable**
   - **Security group:** Create a new one:
     - **Name:** `ssm-instance-sg`
     - **Inbound rules:** **none** (leave empty — delete the default SSH rule if present)
     - **Outbound rules:** leave default (Allow all outbound) — the agent needs outbound HTTPS (443)
7. Expand **Advanced details**:
   - **IAM instance profile:** select `EC2-SSM-Role`
8. Leave storage as default
9. Click **Launch instance**
10. Wait until **Instance state = Running** and **Status checks = 2/2 checks passed**

---

## 8. Step 3: Set Up VPC Interface Endpoints (Required for Private Subnet — No NAT/IGW)

Do this **before** checking Fleet Manager. Since `ssm-private-subnet` has **no route to the internet** (no NAT Gateway, no IGW), the SSM Agent cannot reach the public SSM service endpoints — and it never will, no matter how long you wait, until this step is done. We fix this using **AWS PrivateLink / VPC Interface Endpoints**, which place SSM's endpoints directly inside your VPC.

**Three endpoints are required together** (all three, not just one):

| Endpoint | Purpose |
|---|---|
| `com.amazonaws.<region>.ssm` | Core Systems Manager API calls |
| `com.amazonaws.<region>.ssmmessages` | Session Manager's actual data channel (the shell traffic) |
| `com.amazonaws.<region>.ec2messages` | Agent-to-service messaging used by SSM |

### 8.1 Create the Endpoint Security Group

1. Go to **EC2 console** → **Security Groups** → **Create security group**
2. **Name:** `ssm-endpoint-sg`
3. **VPC:** `ssm-lab-vpc`
4. **Inbound rules:**
   - Type: **HTTPS**, Port: **443**, Source: `ssm-instance-sg` (select the security group, not an IP range)
5. **Outbound rules:** leave default (allow all)
6. Click **Create security group**

### 8.2 Create the Three Interface Endpoints

Repeat this three times, once per service name.

1. Go to **VPC console** → **Endpoints** → **Create endpoint**
2. **Name tag:** `vpce-ssm` (then `vpce-ssmmessages`, then `vpce-ec2messages` on the next two runs)
3. **Service category:** AWS services
4. **Service name:** search and select:
   - Run 1: `com.amazonaws.<your-region>.ssm`
   - Run 2: `com.amazonaws.<your-region>.ssmmessages`
   - Run 3: `com.amazonaws.<your-region>.ec2messages`
5. **VPC:** `ssm-lab-vpc`
6. **Subnets:** select the AZ containing `ssm-private-subnet` (check the box for that AZ)
7. **Security groups:** uncheck default, check `ssm-endpoint-sg`
8. **Policy:** Full access (default)
9. Click **Create endpoint**
10. Repeat for the remaining two service names

> Wait until all three endpoints show **Status: Available** (takes 1–3 minutes each) before proceeding to Step 4.

### 8.3 Confirm DNS Resolution Is Enabled

1. Select `ssm-lab-vpc` in the VPC console
2. **Actions** → **Edit VPC settings**
3. Ensure **Enable DNS resolution** and **Enable DNS hostnames** are both checked
4. Save if you changed anything

This step matters because the SSM Agent resolves the SSM service hostname, and that hostname must resolve to the **private** endpoint IP (not a public IP) inside the VPC — which only works correctly when DNS resolution is enabled on the VPC.

> **If you're on Scenario A** (your subnet already has a NAT Gateway or IGW route), skip this entire Step 3 — go straight to Step 4.

---

## 9. Step 4: Verify SSM Agent Is Installed & Running

Amazon Linux 2023, Amazon Linux 2, and Ubuntu 20.04+ AMIs from AWS come with the SSM Agent **pre-installed**. You do not need to install it manually on these AMIs. Now that the endpoints exist (or you're on a subnet with NAT/IGW), the agent has a real path to reach SSM — this is the point where it's actually meaningful to check.

### 9.1 Check via Systems Manager Console (Fleet Manager)

1. Go to **Systems Manager console** → left sidebar → **Fleet Manager** (under Node Management)
2. Look for `ssm-private-instance` in the managed instances list
3. **Ping status** should show **Online** (allow 1–2 minutes after endpoints become Available)

> If it's still not appearing after a few minutes, that's now a real issue, not an expected wait — check the **Troubleshooting Checklist** section.

### 9.2 (If Needed) Manually Verify/Install/Restart the Agent on the Instance Itself

You will only need this section if:
- You used a custom/older AMI where the agent isn't pre-installed, OR
- Fleet Manager still shows the instance as offline after the endpoints are Available, and you need to debug from inside

Since the instance has no SSH/public IP, you can only run these commands **after** you already have Session Manager access (chicken-and-egg only applies to brand-new custom AMIs — standard Amazon Linux/Ubuntu AMIs already have it pre-installed and running, so Step 5 below will just work once it's Online).

**Check agent status (Amazon Linux 2023 / Amazon Linux 2 / RHEL):**
```bash
sudo systemctl status amazon-ssm-agent
```

**Check agent status (Ubuntu):**
```bash
sudo systemctl status snap.amazon-ssm-agent.amazon-ssm-agent.service
```

**Start the agent if stopped:**
```bash
sudo systemctl start amazon-ssm-agent
sudo systemctl enable amazon-ssm-agent
```

**Manually install the agent (Amazon Linux, if genuinely missing):**
```bash
sudo yum install -y amazon-ssm-agent
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
```

**Manually install the agent (Ubuntu, if genuinely missing):**
```bash
sudo snap install amazon-ssm-agent --classic
sudo snap start amazon-ssm-agent
```

**Check the agent's version:**
```bash
sudo systemctl status amazon-ssm-agent | grep "Active"
amazon-ssm-agent -version
```

**View agent logs (for debugging registration issues):**
```bash
sudo tail -f /var/log/amazon/ssm/amazon-ssm-agent.log
```

---

## 10. Step 5: Connect to the Private Instance Using Session Manager

Now the actual login.

1. Go to **EC2 console** → **Instances**
2. Select the checkbox next to `ssm-private-instance`
3. Click **Connect** (top right)
4. In the **Connect to instance** page, select the **Session Manager** tab
5. You should see: *"Amazon EC2 Instance Connect and Session Manager"* with your instance listed and eligible
6. Click **Connect**

A new browser tab opens with a **black terminal window** — you are now inside the private EC2 instance's shell, with:

- No SSH client used
- No public IP on the instance
- No inbound security group rule
- No key pair

You'll land in a shell as the `ssm-user` (a user automatically created by the SSM Agent for session access).

---

## 11. Step 6: Verify the Session (Prove It's Actually Private)

Run these inside the Session Manager terminal you just opened, to confirm you're truly on a private instance with no direct internet path:

**Confirm no public IP is attached:**
```bash
curl -s http://169.254.169.254/latest/meta-data/public-ipv4 ; echo
```
Expected: empty output (no public IP exists).

**Confirm the private IP:**
```bash
curl -s http://169.254.169.254/latest/meta-data/local-ipv4 ; echo
```
Expected: an address inside `10.0.1.0/24`.

**Confirm you are the ssm-user:**
```bash
whoami
```
Expected output: `ssm-user`

**Confirm the SSM Agent is what's serving this session:**
```bash
sudo systemctl status amazon-ssm-agent
```
Expected: `Active: active (running)`

**Confirm there is genuinely no internet route (optional, proves the PrivateLink path):**
```bash
curl -m 5 https://www.google.com
```
Expected: this should **time out or fail** — proving the instance has no internet path, yet Session Manager still worked, because it used the VPC Interface Endpoints, not the public internet.

To end the session, either close the browser tab or type:
```bash
exit
```

---

## 12. CLI Commands Reference (Equivalent Actions)

These are provided as a **reference only** — this lab was designed to be done via console. Use these if you later want to script the same workflow.

**Start a session (from your local machine, requires Session Manager plugin installed):**
```bash
aws ssm start-session --target i-0123456789abcdef0
```

**List all managed instances (i.e., agent successfully registered):**
```bash
aws ssm describe-instance-information
```

**Check a specific instance's ping status:**
```bash
aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=i-0123456789abcdef0"
```

**Send a command without an interactive session (Run Command):**
```bash
aws ssm send-command \
  --instance-ids "i-0123456789abcdef0" \
  --document-name "AWS-RunShellScript" \
  --parameters commands="sudo systemctl status amazon-ssm-agent"
```

**Terminate an active session:**
```bash
aws ssm terminate-session --session-id <session-id>
```

**List active sessions:**
```bash
aws ssm describe-sessions --state Active
```

---

## 13. Terraform Code (Infrastructure as Code)

```hcl
provider "aws" {
  region = "ap-south-1"
}

# VPC
resource "aws_vpc" "ssm_lab_vpc" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "ssm-lab-vpc" }
}

# Private Subnet (no IGW/NAT route)
resource "aws_subnet" "ssm_private_subnet" {
  vpc_id            = aws_vpc.ssm_lab_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "ap-south-1a"
  tags = { Name = "ssm-private-subnet" }
}

# IAM Role for SSM
resource "aws_iam_role" "ec2_ssm_role" {
  name = "EC2-SSM-Role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ssm_core_attach" {
  role       = aws_iam_role.ec2_ssm_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_instance_profile" "ec2_ssm_profile" {
  name = "EC2-SSM-Profile"
  role = aws_iam_role.ec2_ssm_role.name
}

# Security Group for the instance — no inbound rules
resource "aws_security_group" "ssm_instance_sg" {
  name   = "ssm-instance-sg"
  vpc_id = aws_vpc.ssm_lab_vpc.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Security Group for VPC endpoints — allow HTTPS from instance SG only
resource "aws_security_group" "ssm_endpoint_sg" {
  name   = "ssm-endpoint-sg"
  vpc_id = aws_vpc.ssm_lab_vpc.id

  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [aws_security_group.ssm_instance_sg.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# VPC Interface Endpoints
resource "aws_vpc_endpoint" "ssm" {
  vpc_id              = aws_vpc.ssm_lab_vpc.id
  service_name        = "com.amazonaws.ap-south-1.ssm"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = [aws_subnet.ssm_private_subnet.id]
  security_group_ids  = [aws_security_group.ssm_endpoint_sg.id]
  private_dns_enabled = true
}

resource "aws_vpc_endpoint" "ssmmessages" {
  vpc_id              = aws_vpc.ssm_lab_vpc.id
  service_name        = "com.amazonaws.ap-south-1.ssmmessages"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = [aws_subnet.ssm_private_subnet.id]
  security_group_ids  = [aws_security_group.ssm_endpoint_sg.id]
  private_dns_enabled = true
}

resource "aws_vpc_endpoint" "ec2messages" {
  vpc_id              = aws_vpc.ssm_lab_vpc.id
  service_name        = "com.amazonaws.ap-south-1.ec2messages"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = [aws_subnet.ssm_private_subnet.id]
  security_group_ids  = [aws_security_group.ssm_endpoint_sg.id]
  private_dns_enabled = true
}

# Always resolve the latest Amazon Linux 2023 AMI dynamically via SSM Parameter Store
# instead of hardcoding an AMI ID (AMI IDs change per region and are rotated by AWS regularly)
data "aws_ssm_parameter" "al2023_ami" {
  name = "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64"
}

# EC2 Instance — no public IP, no key pair
resource "aws_instance" "ssm_private_instance" {
  ami                         = data.aws_ssm_parameter.al2023_ami.value
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.ssm_private_subnet.id
  vpc_security_group_ids      = [aws_security_group.ssm_instance_sg.id]
  iam_instance_profile        = aws_iam_instance_profile.ec2_ssm_profile.name
  associate_public_ip_address = false

  tags = { Name = "ssm-private-instance" }
}
```

---

## 14. Edge Cases & Failure Scenarios

| Scenario | What Happens | Root Cause | Fix |
|---|---|---|---|
| Instance not visible in Fleet Manager | "Managed instance" never appears | IAM role missing/wrong, or agent not running | Attach `AmazonSSMManagedInstanceCore` to instance role; verify agent status |
| Fleet Manager shows "Connection lost" | Was online, now offline | Agent crashed, instance stopped/started with role detached, or network path broken | Restart agent; check instance profile is still attached; check endpoint health |
| "Connect" button greyed out / Session Manager tab missing | Console won't let you start a session | No IAM role attached at all, or agent never registered | Attach IAM role; may require instance reboot for role attachment to take effect in some agent versions |
| Session starts but immediately terminates | Session opens then closes in 1–2 seconds | SSM document `SSM-SessionManagerRunShell` missing/misconfigured, or agent version too old | Update SSM Agent; check Session Manager preferences under Systems Manager → Session Manager → Preferences |
| Private subnet + no VPC endpoints created | Instance never registers as "Online" | No path to reach SSM service (no NAT/IGW and no PrivateLink) | Create all 3 required interface endpoints (ssm, ssmmessages, ec2messages) |
| Endpoints created but instance still offline | Same as above, persists | Endpoint security group doesn't allow inbound 443 from instance's SG, or endpoint deployed in wrong AZ/subnet | Fix endpoint SG inbound rule; ensure endpoint subnet matches instance's AZ |
| DNS resolution disabled on VPC | Endpoints exist but agent can't find them | VPC-level "Enable DNS resolution" or "Enable DNS hostnames" turned off | Enable both under VPC settings |
| Old/custom AMI without SSM Agent | Instance boots, but never appears in Fleet Manager | Agent was never installed at image build time | Bake agent into custom AMI at build time, or install via user-data script at launch |
| IAM role attached to instance AFTER launch | Instance still not appearing | Attaching a role after launch sometimes requires the agent to restart to pick up new credentials | Reboot instance, or (if you already have a session another way) restart the agent manually |
| Session Manager logging required for compliance | No record of what commands were run | Session logging wasn't enabled | Enable session logging to S3/CloudWatch Logs under Session Manager → Preferences |
| Multiple users need session access | Uncontrolled access across team | No IAM policy scoping who can start sessions on which instances | Use IAM policies with `ssm:StartSession` scoped by tag condition keys |
| VPC endpoint in wrong VPC / peered VPC | Instance inVpc-A, endpoint in Vpc-B | Interface endpoints are VPC-scoped; PrivateLink doesn't traverse VPC peering by default for endpoint DNS resolution the same way | Endpoint must exist in the same VPC as the instance (or reachable via endpoint-specific sharing/Resource Access Manager) |

---

## 15. Troubleshooting Checklist

Run through this top-to-bottom whenever "Connect via Session Manager" doesn't work:

1. ☐ Is the IAM role attached to the instance, and does it include `AmazonSSMManagedInstanceCore`?
2. ☐ Is the SSM Agent installed on the AMI? (Amazon Linux 2/2023, Ubuntu 20.04+ have it by default)
3. ☐ Is the agent actually running? (`systemctl status amazon-ssm-agent`)
4. ☐ Does the instance show **Online** in Fleet Manager?
5. ☐ If the subnet has no NAT/IGW — do all 3 required VPC interface endpoints exist and show **Available**?
6. ☐ Does the endpoint security group allow inbound **443** from the instance's security group?
7. ☐ Is **DNS resolution** and **DNS hostnames** enabled on the VPC?
8. ☐ Is **private DNS enabled** on the VPC endpoints? (should be checked by default)
9. ☐ Is the instance's outbound security group rule allowing **443 outbound**? (default "allow all outbound" covers this)
10. ☐ Check agent logs on the instance (once you get any access): `/var/log/amazon/ssm/amazon-ssm-agent.log`

---

## 16. Best Practices

- **Never open port 22/3389** on production instances just "in case" — Session Manager replaces the need entirely.
- **Scope IAM permissions** for `ssm:StartSession` using tag-based conditions so engineers can only start sessions on instances they own.
- **Enable session logging** to S3 and/or CloudWatch Logs for audit/compliance — this gives you a full transcript of every session.
- **Enable CloudTrail logging** for SSM API calls (`StartSession`, `TerminateSession`) to track who connected and when at the account level.
- **Use Session Manager run-as support** to map sessions to a specific OS user instead of the default `ssm-user`, for better audit trails.
- **Rotate/patch the SSM Agent** — AWS periodically releases updates; keep agents current via Patch Manager or a maintenance window.
- **Use VPC endpoints for all Systems Manager traffic** in private/air-gapped environments — never punch a NAT Gateway hole just for SSM if you can avoid it, since it's an unnecessary cost and attack surface.
- **Tag your endpoints and instances consistently** so IAM tag-based conditions and cost allocation both work cleanly.
- **Disable Session Manager idle timeout carefully** — configure it under Session Manager Preferences rather than leaving default indefinite sessions open.

---

## 17. Interview Questions & Answers

**Q1: How does Session Manager access a private EC2 instance without SSH or a bastion host?**
A: The SSM Agent on the instance initiates an outbound HTTPS connection to the Systems Manager service (either directly or via VPC Interface Endpoints if there's no internet route). When a session is started from the console/CLI, Systems Manager brokers a secure channel over that existing outbound connection — no inbound port is ever opened on the instance.

**Q2: What IAM policy is minimally required for an EC2 instance to work with Session Manager?**
A: The AWS managed policy `AmazonSSMManagedInstanceCore`, attached to an IAM role that's set as the instance's IAM instance profile.

**Q3: My instance is in a fully private subnet with no NAT Gateway. Will Session Manager still work?**
A: Yes, but only if you create three VPC Interface Endpoints in that subnet's VPC: `ssm`, `ssmmessages`, and `ec2messages`, each with a security group allowing inbound HTTPS (443) from the instance's security group, and with DNS resolution enabled on the VPC.

**Q4: What's the difference between `ssm`, `ssmmessages`, and `ec2messages` endpoints?**
A: `ssm` handles core Systems Manager API calls (like instance registration/inventory); `ec2messages` is used for the agent to poll for commands/messages; `ssmmessages` carries the actual interactive session data channel used by Session Manager specifically.

**Q5: How would you audit who accessed a production instance and what they typed?**
A: Enable Session Manager logging (to S3 and/or CloudWatch Logs) under Session Manager Preferences, and enable CloudTrail for `ssm:StartSession`/`TerminateSession` API calls. Together these give both the session transcript and the "who/when" audit trail.

**Q6: Why might an instance not appear as "Managed" in Fleet Manager even though it's running?**
A: Most commonly: no IAM role attached (or missing the SSM managed policy), the SSM Agent isn't running/installed, or — in a private subnet — the required VPC interface endpoints don't exist so the agent can never register with the service.

**Q7: Can you restrict which engineers can start a session on which instances?**
A: Yes — using IAM policy conditions on `ssm:StartSession` scoped to resource tags (e.g., `ssm:resourceTag/Environment: dev`), so an engineer's IAM policy only allows sessions on instances carrying specific tags.

**Q8: Does Session Manager require the instance to have a public IP?**
A: No. Session Manager works entirely through the agent's outbound connection; a public IP is never required, and in the private-subnet pattern the instance has no public IP at all.

---

## 18. Cheat Sheet

```
Required IAM Policy:        AmazonSSMManagedInstanceCore
Agent status (AL2/AL2023):  sudo systemctl status amazon-ssm-agent
Agent status (Ubuntu):      sudo systemctl status snap.amazon-ssm-agent.amazon-ssm-agent.service
Agent logs:                 /var/log/amazon/ssm/amazon-ssm-agent.log
Console check-in point:     Systems Manager → Fleet Manager → Ping status = Online
Connect (console):          EC2 → Instances → select → Connect → Session Manager tab → Connect
Connect (CLI):              aws ssm start-session --target <instance-id>
Required endpoints          com.amazonaws.<region>.ssm
(private subnet only):      com.amazonaws.<region>.ssmmessages
                             com.amazonaws.<region>.ec2messages
Endpoint SG rule:            Inbound 443 from instance security group
VPC requirement:             DNS resolution + DNS hostnames enabled
Default session user:        ssm-user
No public IP needed:         True
No inbound SG rule needed:   True
No SSH key pair needed:      True
```

---

## 19. Mastery Checklist

- [ ] I can explain why Session Manager doesn't need inbound ports or a bastion host
- [ ] I can create an IAM role with `AmazonSSMManagedInstanceCore` and attach it to an EC2 instance
- [ ] I can verify SSM Agent status and read its logs on the instance
- [ ] I can identify when VPC Interface Endpoints are required (no NAT/IGW) versus not required
- [ ] I can create all three required interface endpoints with correctly scoped security groups
- [ ] I can launch a fully private EC2 instance (no public IP, no key pair) and successfully connect via Session Manager
- [ ] I can prove the instance has no internet path while still successfully using Session Manager
- [ ] I can troubleshoot an instance that isn't appearing as "Online" in Fleet Manager
- [ ] I can explain how to enable and where to find session audit logs
- [ ] I can write the Terraform to reproduce this entire setup as code
