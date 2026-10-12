# EC2 Instance Connect Endpoint (EICE): End-to-End Setup Guide

Console-based, no steps skipped. Follow in order.

---

## 1. What You Are Building

**EC2 Instance Connect Endpoint (EICE)** lets you SSH (or RDP) into EC2 instances that are in a **private subnet with no public IP**, with:

- No bastion host
- No Internet Gateway required
- No public IP on the instance
- Access controlled by IAM + Security Groups
- Connection attempts logged in CloudTrail

### Architecture

```
You (browser / local terminal)
        |
        |  HTTPS (AWS API, IAM-authenticated)
        v
EC2 Instance Connect Endpoint  (lives in a private subnet, has an ENI + Security Group)
        |
        |  TCP 22 (private traffic inside the VPC)
        v
Private EC2 instance  (no public IP)
```

### How the connection works

1. You are authenticated with IAM.
2. AWS opens a tunnel through the endpoint to the instance's private IP.
3. For Linux, EC2 Instance Connect pushes a **temporary SSH public key** (valid ~60 seconds) to the instance, so you do not need to manage a key pair for the session.

---

## 2. Prerequisites

- An AWS account with permission to create VPC, EC2, and IAM resources (an admin user is easiest for this lab).
- A region to work in. This guide uses **ap-south-1 (Mumbai)**. Use any region, but stay in the same one for every step.
- A supported instance OS: **Amazon Linux 2023**, Amazon Linux 2, or Ubuntu 20.04+. These have EC2 Instance Connect installed by default. This guide uses **Amazon Linux 2023**.

> **Cost note:** The endpoint itself has no hourly charge. Standard data transfer charges apply. The EC2 instance is charged normally. Clean up at the end (Section 11).

---

## 3. Step 1: Create a VPC With a Private Subnet Only

If you already have a VPC with a private subnet, skip to Step 2 and use that.

1. Sign in to the **AWS Management Console**.
2. In the top-right, confirm the **Region** (for example, Asia Pacific (Mumbai) `ap-south-1`).
3. In the search bar, type **VPC** and open **VPC**.
4. In the left menu, click **Your VPCs**.
5. Click **Create VPC**.
6. Under **Resources to create**, select **VPC and more**.
7. Set the following:

| Setting | Value |
|---|---|
| Name tag auto-generation | Keep **Auto-generate** checked, enter `eice-lab` |
| IPv4 CIDR block | `10.0.0.0/16` |
| IPv6 CIDR block | No IPv6 CIDR block |
| Tenancy | Default |
| Number of Availability Zones (AZs) | **1** |
| Number of public subnets | **0** |
| Number of private subnets | **1** |
| NAT gateways | **None** |
| VPC endpoints | **None** |
| Enable DNS hostnames | Checked |
| Enable DNS resolution | Checked |

8. Click **Create VPC**.
9. Wait until all items show a green check, then click **View VPC**.
10. Note down:
    - VPC ID (for example, `vpc-0abc123...`)
    - Private subnet name (for example, `eice-lab-subnet-private1-ap-south-1a`)

> With no public subnet and no NAT, the instance has **zero** internet path. That is exactly what proves EICE works.

---

## 4. Step 2: Create the Security Groups

You need **two** Security Groups:

- `eice-instance-sg` attached to the EC2 instance
- `eice-endpoint-sg` attached to the endpoint

Create the instance SG first so the endpoint SG can reference it.

### 4.1 Create the instance Security Group

1. Open the **EC2** console.
2. In the left menu, under **Network & Security**, click **Security Groups**.
3. Click **Create security group**.
4. Fill in:
   - **Security group name:** `eice-instance-sg`
   - **Description:** `Security group for private EC2 instance reached via EICE`
   - **VPC:** select `eice-lab-vpc`
5. **Inbound rules:** leave empty for now (you add one in 4.3).
6. **Outbound rules:** leave the default (All traffic).
7. Click **Create security group**.

### 4.2 Create the endpoint Security Group

1. Click **Create security group** again.
2. Fill in:
   - **Security group name:** `eice-endpoint-sg`
   - **Description:** `Security group for EC2 Instance Connect Endpoint`
   - **VPC:** select `eice-lab-vpc`
3. **Inbound rules:** leave empty. The endpoint does not need any inbound rules.
4. **Outbound rules:**
   1. Delete the default **All traffic** rule by clicking **Delete** next to it.
   2. Click **Add rule**.
   3. **Type:** `SSH`, **Protocol:** TCP, **Port range:** `22`
   4. **Destination:** choose **Custom**, then select `eice-instance-sg` from the dropdown.
5. Click **Create security group**.

### 4.3 Allow SSH from the endpoint to the instance

1. In **Security Groups**, select `eice-instance-sg`.
2. Open the **Inbound rules** tab, click **Edit inbound rules**.
3. Click **Add rule**:
   - **Type:** `SSH`
   - **Port range:** `22`
   - **Source:** choose **Custom**, then select `eice-endpoint-sg`
   - **Description:** `SSH from EICE`
4. Click **Save rules**.

> **Rule of thumb:** the instance SG must **allow inbound 22 from the endpoint SG**, and the endpoint SG must **allow outbound 22 to the instance SG**. If either is missing, the connection fails.

---

## 5. Step 3: Give Your IAM Identity the Required Permissions

If you are using an **Administrator** user or role, skip to Step 4.

Otherwise, create and attach a policy.

### 5.1 Permissions to create the endpoint

Your IAM identity needs:

- `ec2:CreateInstanceConnectEndpoint`
- `ec2:CreateNetworkInterface`
- `ec2:CreateTags`
- `iam:CreateServiceLinkedRole`
- `ec2:DescribeInstanceConnectEndpoints`
- `ec2:DescribeSubnets`, `ec2:DescribeSecurityGroups`, `ec2:DescribeVpcs`

### 5.2 Permissions to connect through the endpoint

1. Open the **IAM** console.
2. Click **Policies**, then **Create policy**.
3. Select the **JSON** tab and paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "UseEICE",
      "Effect": "Allow",
      "Action": [
        "ec2-instance-connect:OpenTunnel",
        "ec2-instance-connect:SendSSHPublicKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DescribeForConsole",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceConnectEndpoints",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DescribeSubnets",
        "ec2:DescribeVpcs"
      ],
      "Resource": "*"
    }
  ]
}
```

4. Click **Next**.
5. **Policy name:** `eice-connect-policy`, then click **Create policy**.
6. Go to **Users** (or **Roles**), select your identity, then **Add permissions** > **Attach policies directly**.
7. Search `eice-connect-policy`, tick it, click **Next**, then **Add permissions**.

> `Resource: "*"` is fine for a lab. In production, restrict `OpenTunnel` to a specific endpoint ARN and use the `ec2-instance-connect:maxTunnelDuration` condition key.

---

## 6. Step 4: Create the EC2 Instance Connect Endpoint

1. Open the **VPC** console.
2. In the left menu, click **Endpoints**.
3. Click **Create endpoint**.
4. Fill in:
   - **Name tag:** `eice-lab-endpoint`
   - **Service category:** select **EC2 Instance Connect Endpoint**
5. **VPC:** select `eice-lab-vpc`.
6. **Security groups:** select `eice-endpoint-sg`.
7. **Subnet:** select your private subnet (for example, `eice-lab-subnet-private1-ap-south-1a`).
8. Leave **Preserve Client IP** unchecked. (Leave it off so the instance sees the endpoint's private IP as the source, which matches the SG rule from Step 2.)
9. Click **Create endpoint**.
10. Wait for **Status** to change from `Pending` / `Creating` to **Available**. This takes a few minutes. Refresh the list until you see **Available**.

> Create only **one endpoint per subnet**. The endpoint can reach any instance in the same VPC whose route and SG allow it.

---

## 7. Step 5: Launch a Private EC2 Instance

1. Open the **EC2** console and click **Instances**.
2. Click **Launch instances**.
3. **Name:** `eice-private-instance`.
4. **Application and OS Images:** select **Amazon Linux** and choose **Amazon Linux 2023 AMI**.
5. **Instance type:** `t3.micro` (or any free-tier eligible type).
6. **Key pair (login):** select **Proceed without a key pair**. EC2 Instance Connect uses a temporary key, so no permanent key is required.
   - If you prefer a key pair for later use, choose or create one. It is optional.
7. Click **Edit** next to **Network settings** and set:
   - **VPC:** `eice-lab-vpc`
   - **Subnet:** your private subnet
   - **Auto-assign public IP:** **Disable**
   - **Firewall (security groups):** select **Select existing security group** and choose `eice-instance-sg`
8. **Configure storage:** keep the default.
9. Click **Launch instance**.
10. Click the instance ID link, then wait until:
    - **Instance state:** `Running`
    - **Status check:** `2/2 checks passed`
11. Confirm in the details pane that **Public IPv4 address** is empty (`-`) and a **Private IPv4 address** (for example, `10.0.x.x`) is shown.

---

## 8. Step 6: Connect Using the Console

1. In **EC2 > Instances**, select `eice-private-instance`.
2. Click **Connect** (top right).
3. Open the **EC2 Instance Connect** tab.
4. For **Connection type**, select **Connect using EC2 Instance Connect Endpoint**.
5. For **EC2 Instance Connect Endpoint**, choose `eice-lab-endpoint`.
6. **Username:** `ec2-user` (Amazon Linux). For Ubuntu use `ubuntu`.
7. Leave **Max tunnel duration** at its default (up to 1 hour).
8. Click **Connect**.
9. A browser terminal opens with a prompt like:

```
[ec2-user@ip-10-0-x-x ~]$
```

You are now inside a private instance with no public IP and no internet path.

---

## 9. Step 7: Verify

Run these inside the browser terminal:

```bash
whoami
hostname
ip addr show
curl -m 5 https://aws.amazon.com
```

Expected results:

| Command | Expected |
|---|---|
| `whoami` | `ec2-user` |
| `hostname` | `ip-10-0-x-x.<region>.compute.internal` |
| `ip addr show` | Only a private `10.0.x.x` address |
| `curl -m 5 https://aws.amazon.com` | Times out (no internet path, which is correct) |

---

## 10. Optional: Connect From Your Local Terminal

This uses the AWS CLI on your laptop. The rest of this guide is console-only; this section is optional.

### Prerequisites

- AWS CLI **v2.12.0 or later**
- Credentials configured (`aws configure`) for the identity that has `eice-connect-policy`

### Option A: One-command SSH

```bash
aws ec2-instance-connect ssh \
  --instance-id i-0123456789abcdef0 \
  --os-user ec2-user \
  --connection-type eice
```

Replace the instance ID with yours.

### Option B: Open a tunnel, then SSH (needs a key pair on the instance)

Terminal 1:

```bash
aws ec2-instance-connect open-tunnel \
  --instance-id i-0123456789abcdef0 \
  --local-port 2222
```

Terminal 2:

```bash
ssh -i my-key.pem -p 2222 ec2-user@localhost
```

---

## 11. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Endpoint option is empty in the Connect page | Endpoint not **Available** yet, or wrong region/VPC | Wait for **Available**; confirm the same region and VPC |
| `Failed to connect to your instance` | SG rules missing or wrong | Instance SG inbound 22 from `eice-endpoint-sg`; endpoint SG outbound 22 to `eice-instance-sg` |
| `Access denied` / `not authorized` | Missing IAM actions | Re-check `eice-connect-policy` is attached; needs `OpenTunnel` and `SendSSHPublicKey` |
| Connect works but login fails | Wrong username | `ec2-user` for Amazon Linux, `ubuntu` for Ubuntu |
| Login fails on a custom AMI | EC2 Instance Connect package missing | Install the `ec2-instance-connect` package on the instance |
| Instance has a NACL | NACL blocks 22 or ephemeral ports | Allow TCP 22 inbound and ephemeral ports outbound for the subnet |
| Instance not in the endpoint's VPC | EICE only reaches instances in its own VPC | Create an endpoint in that VPC |

---

## 12. Cleanup (Avoid Charges)

Delete in this order:

1. **EC2 > Instances:** select `eice-private-instance` > **Instance state** > **Terminate instance** > confirm. Wait for `Terminated`.
2. **VPC > Endpoints:** select `eice-lab-endpoint` > **Actions** > **Delete VPC endpoints** > type `delete` > confirm. Wait until it is gone.
3. **EC2 > Security Groups:**
   1. Select `eice-endpoint-sg` and **Delete security groups**. If it errors, first remove the reference to it in `eice-instance-sg`'s inbound rule.
   2. Select `eice-instance-sg` and **Delete security groups**.
4. **VPC > Your VPCs:** select `eice-lab-vpc` > **Actions** > **Delete VPC**. This also removes the subnet and route table.
5. **IAM > Policies:** (optional) detach and delete `eice-connect-policy`.

---

## 13. Quick Summary Checklist

- [ ] VPC with a private subnet created
- [ ] `eice-instance-sg`: inbound SSH 22 from `eice-endpoint-sg`
- [ ] `eice-endpoint-sg`: outbound SSH 22 to `eice-instance-sg`, no inbound
- [ ] IAM identity has `OpenTunnel` + `SendSSHPublicKey`
- [ ] Endpoint created and status **Available**
- [ ] EC2 instance in private subnet, no public IP, Amazon Linux 2023
- [ ] Connected via **EC2 Instance Connect Endpoint** in the console
- [ ] Resources cleaned up
