# AWS Hands-on Lab: Host a Website on a Private EC2 Instance Behind an Application Load Balancer (ALB)

**Difficulty:** Intermediate
**Estimated Time:** 75–100 Minutes

**AWS Services Used:**
- VPC
- Internet Gateway
- Route Tables
- Security Groups
- EC2 (Bastion Host + Private Web Server)
- Application Load Balancer (ALB)
- Target Group
- Amazon Linux 2023

**Connectivity Method:** Bastion Host (SSH Jump Box)

---

## Architecture

```
                          Internet
                              │
                        HTTP (80)
                              │
                ┌────────────────────────┐
                │ Application Load Balancer│
                │      Public Subnets      │
                └────────────┬─────────────┘
                             │
                     Target Group
                             │
                             ▼
                  Private EC2 Instance
                     Apache Website
                             │
                Private Subnet (No Public IP)
                             │
                      Route Table (No IGW)

   Admin Access Path (separate from web traffic):

   You (SSH) ──▶ Bastion Host ──▶ Private EC2
                (Public Subnet)    (Private Subnet)
                (Public IP)        (No Public IP)
```

## Final Architecture

```
                    +----------------------+
                    |      Internet        |
                    +----------+-----------+
                               │
                 ┌─────────────┴─────────────┐
                 │                            │
          HTTP/HTTPS Request              SSH (22)
                 │                            │
                 ▼                            ▼
   +--------------------------------+   +------------------+
   | Application Load Balancer (ALB)|   |  Bastion Host    |
   | Public Subnet A & Public B     |   |  Public Subnet A |
   +---------------+----------------+   +--------+---------+
                    │                            │
             Target Group                    SSH (22)
                    │                            │
              Health Check (/)                   │
                    │                            │
                    ▼                            ▼
       +-------------------------+     (same instance)
       | Private EC2 Instance    |◀────────────┘
       | Apache Website          |
       | No Public IP            |
       +-------------------------+
                    │
             Private Route Table
                    │
              Local VPC Traffic Only
```

---

## Prerequisites

- AWS Account
- IAM User with EC2 permissions
- Key Pair (same key pair can be used for both Bastion and Private EC2, or use two separate ones)
- Region Selected (Mumbai used in this guide)

---

## Step 1 — Create VPC

Navigate to **VPC** → Click **Create VPC**

| Parameter | Value       |
| --------- | ----------- |
| Name      | Website-VPC |
| IPv4 CIDR | 10.0.0.0/16 |
| IPv6      | None        |

Create VPC.

---

## Step 2 — Create Internet Gateway

Go to **Internet Gateway** → Create `Website-IGW`

After creation, attach it to `Website-VPC`.

---

## Step 3 — Create Public Subnets

Create two public subnets because ALB requires at least two Availability Zones. The Bastion Host will also live in a public subnet.

**Public Subnet 1**

| Parameter | Value           |
| --------- | --------------- |
| Name      | Public-Subnet-1 |
| AZ        | ap-south-1a     |
| CIDR      | 10.0.1.0/24     |

**Public Subnet 2**

| Parameter | Value           |
| --------- | --------------- |
| Name      | Public-Subnet-2 |
| AZ        | ap-south-1b     |
| CIDR      | 10.0.2.0/24     |

Enable **Auto Assign Public IPv4** for both public subnets.

---

## Step 4 — Create Private Subnet

| Parameter | Value          |
| --------- | -------------- |
| Name      | Private-Subnet |
| AZ        | ap-south-1a    |
| CIDR      | 10.0.10.0/24   |

Leave **Auto Assign Public IP = Disabled**.

---

## Step 5 — Create Route Tables

### Public Route Table

Create `Public-RT`

| Destination | Target           |
| ----------- | ---------------- |
| 10.0.0.0/16 | Local            |
| 0.0.0.0/0   | Internet Gateway |

Associate:
- Public Subnet 1
- Public Subnet 2

### Private Route Table

Create `Private-RT`

| Destination | Target |
| ----------- | ------ |
| 10.0.0.0/16 | Local  |

Associate: `Private Subnet`

> **Notice:** No Internet Gateway route. The private instance stays fully isolated from the internet — the Bastion only gives you SSH access, it does not give the private instance outbound internet access. (See the NAT Gateway note in Troubleshooting if you need `dnf install` to work.)

---

## Step 6 — Create Security Groups

Three security groups are needed: one for the ALB, one for the Bastion Host, and one for the private web server.

### ALB Security Group

Name: `ALB-SG`

**Inbound**

| Type | Port | Source    |
| ---- | ---- | --------- |
| HTTP | 80   | 0.0.0.0/0 |

*(Optional)*

| Type  | Port | Source    |
| ----- | ---- | --------- |
| HTTPS | 443  | 0.0.0.0/0 |

**Outbound:** Allow All

### Bastion Security Group

Name: `Bastion-SG`

**Inbound**

| Type | Port | Source                              |
| ---- | ---- | ------------------------------------ |
| SSH  | 22   | Your IP (`My IP`) — not 0.0.0.0/0    |

> Restrict this to your own IP. An SSH port open to the entire internet is a common breach vector for bastion hosts.

**Outbound:** Allow All

### Private Web Server Security Group

Name: `Private-Web-SG`

**Inbound**

| Type | Port | Source     |
| ---- | ---- | ---------- |
| HTTP | 80   | ALB-SG     |
| SSH  | 22   | Bastion-SG |

**Outbound:** Allow All

---

## Step 7 — Launch Bastion Host

Navigate to **EC2 → Launch Instance**

| Setting               | Value              |
| ---------------------- | ------------------ |
| Name                   | Bastion-Host        |
| AMI                    | Amazon Linux 2023   |
| Instance Type          | t2.micro             |
| Key Pair               | Choose existing      |
| Network (VPC)          | Website-VPC           |
| Subnet                 | Public-Subnet-1        |
| Auto Assign Public IP  | Enabled                 |
| Security Group         | Bastion-SG               |

Launch, then note down the Bastion's **Public IPv4 address** — you'll need it in Step 9.

---

## Step 8 — Launch Private EC2

Navigate to **EC2 → Launch Instance**

| Setting                | Value             |
| ----------------------- | ----------------- |
| Name                    | Private-WebServer |
| AMI                     | Amazon Linux 2023 |
| Instance Type           | t2.micro           |
| Key Pair                | Choose existing (same key as Bastion, or a separate one) |
| Network (VPC)           | Website-VPC        |
| Subnet                  | Private Subnet     |
| Auto Assign Public IP   | Disabled           |
| Security Group          | Private-Web-SG     |

Launch, then verify the instance shows **Private IP Only / No Public IP**. Note down its **Private IPv4 address**.

---

## Step 9 — Connect to Private EC2 via Bastion Host

Don't copy your private key onto the Bastion — if the Bastion is ever compromised, your key would be exposed. Use **SSH agent forwarding** instead, so the key stays on your local machine.

**On your local machine**, add the key to your SSH agent:

```bash
eval "$(ssh-agent -s)"
ssh-add /path/to/your-key.pem
```

**Connect to the Bastion with agent forwarding enabled (`-A`):**

```bash
ssh -A ec2-user@<BASTION_PUBLIC_IP>
```

**From inside the Bastion, hop to the private instance** (no key file needed here — it's forwarded from your local agent):

```bash
ssh ec2-user@<PRIVATE_EC2_PRIVATE_IP>
```

You should now have a shell on the private instance.

> **Alternative (single command from local machine):**
> ```bash
> ssh -A -J ec2-user@<BASTION_PUBLIC_IP> ec2-user@<PRIVATE_EC2_PRIVATE_IP>
> ```
> The `-J` flag uses the Bastion as a "jump host" and connects you straight through in one command.

---

## Step 10 — Install Apache

While connected to the **private EC2 instance** (via Bastion), run:

```bash
sudo dnf update -y
sudo dnf install httpd -y
sudo systemctl enable httpd
sudo systemctl start httpd
systemctl status httpd
```

> If `dnf update` hangs or fails, the private instance has no outbound internet path — see the NAT Gateway note in Troubleshooting.

---

## Step 11 — Create Website

```bash
sudo tee /var/www/html/index.html <<EOF
<!DOCTYPE html>
<html>
<head>
<title>Private Website</title>
<style>
body{
background:#111;
color:white;
text-align:center;
font-family:Arial;
padding-top:120px;
}
h1{
font-size:48px;
color:#00ff99;
}
p{
font-size:24px;
}
</style>
</head>
<body>

<h1>Website Hosted on Private EC2</h1>

<p>Traffic is coming through Application Load Balancer</p>

<p>Private Server</p>

</body>
</html>
EOF
```

Verify locally:

```bash
curl localhost
```

---

## Step 12 — Create Target Group

Navigate to **EC2 → Target Groups** → Create

| Parameter | Value       |
| --------- | ----------- |
| Type      | Instances   |
| Name      | Private-TG  |
| Protocol  | HTTP        |
| Port      | 80          |
| VPC       | Website-VPC |

Health Check path: `/`

Next → Register Target → choose `Private-WebServer` → **Include as Pending** → **Create Target Group**

> **Note:** The target will show as **Unused** at this point — Target Groups only receive health check results once attached to a Load Balancer. This target group isn't attached to anything yet, so health checks won't begin until the ALB is created in Step 13 and status is confirmed in Step 14.

---

## Step 13 — Create Application Load Balancer

Navigate to **Load Balancers** → Create **Application Load Balancer**

| Parameter | Value           |
| --------- | --------------- |
| Name      | Website-ALB     |
| Scheme    | Internet Facing |
| IP Type   | IPv4            |
| Network   | Website-VPC     |
| Subnets   | Public Subnet 1, Public Subnet 2 |
| Security Group | ALB-SG     |
| Listener  | HTTP 80         |
| Default Action | Forward to `Private-TG` |

Create.

---

## Step 14 — Verify Target Health

Open **Target Groups → Targets**. Expected status: **Healthy**

If unhealthy, check:
- Apache running
- Port 80
- Security groups
- Health check path
- EC2 running

---

## Step 15 — Access Website

Copy the **ALB DNS Name**, e.g.:

```
http://website-alb-123456.ap-south-1.elb.amazonaws.com
```

Open it in a browser. Expected output:

```
Website Hosted on Private EC2
Traffic is coming through Application Load Balancer
```

---

## Step 16 — Verify Private EC2

Open the EC2 console and observe **Public IPv4: None**.

Attempting to access the EC2 instance directly over the internet should be impossible — only the ALB (for HTTP) and the Bastion (for SSH) can reach it.

---

## Security Flow

**Web traffic:**

```
Internet
   │
   ▼
ALB SG
Allow HTTP 80
   │
   ▼
Private EC2 SG
Allow HTTP ONLY from ALB SG
   │
   ▼
Apache
```

**Admin/SSH traffic:**

```
Your IP
   │
   ▼
Bastion SG
Allow SSH 22 ONLY from Your IP
   │
   ▼
Bastion Host (Public Subnet)
   │
   ▼
Private EC2 SG
Allow SSH 22 ONLY from Bastion SG
   │
   ▼
Private EC2 (Private Subnet)
```

## Request Flow

```
Browser → DNS → Application Load Balancer → Target Group →
Private EC2 → Apache → Response → ALB → Browser
```

---

## Validation Checklist

| Test                                       | Expected |
| -------------------------------------------- | -------- |
| Private EC2 has Public IP                    | ❌ No     |
| Bastion Host has Public IP                   | ✅ Yes    |
| SSH to Private EC2 directly from internet     | ❌ No     |
| SSH to Private EC2 via Bastion               | ✅ Yes    |
| ALB Accessible                               | ✅ Yes    |
| Website Loads                                | ✅ Yes    |
| Target Healthy                               | ✅ Yes    |
| Apache Running                               | ✅ Yes    |
| Port 80 Open from Internet to EC2            | ❌ No     |
| Port 80 Open from ALB to EC2                 | ✅ Yes    |
| Port 22 Open from Internet to EC2            | ❌ No     |
| Port 22 Open from Bastion to EC2             | ✅ Yes    |

---

## Common Issues & Troubleshooting

| Issue | Possible Cause | Solution |
| ----- | --------------- | -------- |
| ALB returns 503 Service Unavailable | No healthy targets | Verify target group registration and health checks |
| Target status is Unhealthy | Apache not running or wrong health check path | Start Apache and ensure `/` returns HTTP 200 |
| Timeout when opening ALB DNS | ALB security group missing inbound HTTP/HTTPS | Allow port 80 (and 443 if applicable) from `0.0.0.0/0` |
| Health check fails | EC2 security group blocks ALB | Allow HTTP (80) only from `ALB-SG` |
| Website works with `curl localhost` but not through ALB | Firewall or security group issue | Check EC2 SG, ALB SG, and target group port |
| Cannot SSH into Bastion | Bastion SG doesn't allow your current IP | Update Bastion-SG inbound rule to your current public IP (it changes if you're on a dynamic connection) |
| `ssh: connect to host <private-ip> port 22: Connection timed out` (from inside Bastion) | Private-Web-SG doesn't allow SSH from Bastion-SG, or you forgot `-A` | Confirm Private-Web-SG inbound rule sources `Bastion-SG`; reconnect to the Bastion with `ssh -A` |
| `Permission denied (publickey)` when hopping to private instance | Key not forwarded, or wrong key added to agent | Run `ssh-add -l` locally to confirm the key is loaded; reconnect with `ssh -A` |
| Private EC2 cannot install packages (`dnf update` hangs) | No internet access (no NAT Gateway) — the Bastion gives you SSH access, not internet access for the private instance | Add a NAT Gateway in the public subnet with a route from the Private Route Table, or pre-bake the AMI with required packages |

---

## Security Best Practices

- Keep application servers in **private subnets** with **no public IPs**.
- Expose only the **Application Load Balancer** to the internet for HTTP/HTTPS.
- Expose only the **Bastion Host** to the internet for SSH — and lock its inbound rule to your specific IP, not `0.0.0.0/0`.
- Restrict the EC2 security group to accept HTTP traffic **only from the ALB security group** and SSH **only from the Bastion security group**.
- Use **SSH agent forwarding** (`-A`) instead of copying private keys onto the Bastion.
- Consider terminating or stopping the Bastion when not actively used, since it's a standing internet-facing entry point.
- Use **HTTPS (port 443)** with an ACM certificate in production.
- Enable **access logs** for the ALB (stored in Amazon S3).
- Configure **AWS WAF** in front of the ALB to protect against common web attacks.
- Use **Auto Scaling Groups** with the target group for high availability.

---

## Cleanup

To avoid ongoing AWS charges, delete resources in the following order:

1. Delete the Application Load Balancer.
2. Delete the Target Group.
3. Terminate the Private EC2 instance.
4. Terminate the Bastion Host instance.
5. Delete the Security Groups (if no longer in use).
6. Delete the Route Tables (after disassociating them if needed).
7. Delete the Subnets.
8. Detach and delete the Internet Gateway.
9. Delete the VPC.

---

## Learning Outcomes

After completing this lab, you will understand how to:

- Design a secure web architecture using **private EC2 instances**.
- Set up a **Bastion Host** as a controlled SSH entry point into a private subnet.
- Use **SSH agent forwarding** to reach private instances without exposing private keys.
- Create and configure an **Application Load Balancer** across multiple Availability Zones.
- Register instances with a **Target Group** and troubleshoot health checks.
- Implement **security groups** that separate web traffic (via ALB) from admin traffic (via Bastion).
- Deploy a website without exposing backend servers directly to the internet.
- Apply AWS networking best practices for production-ready, highly available web applications.
