# Blue/Green Deployment in AWS

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/9bfebfe3-155d-41da-8099-11e806d4fa51" />


## Part 1: Foundation Network (VPC)

**Goal:** Create the foundation network.

**Steps:**
1. Navigate to VPC Dashboard > Create VPC
2. Configure:
   - Name tag: `network-vpc`
   - IPv4 CIDR block: `10.0.0.0/16`
3. Click **Create VPC**

---

## Part 2: Subnets in Multiple Availability Zones

**Goal:** Create subnets in multiple availability zones.

### Step 1: Create Subnet 1a
- Navigate to Subnets > Create subnet
- Configure:
  - VPC ID: Select `network-vpc`
  - Subnet name: `public-subnet-1a`
  - Availability Zone: `us-east-1a`
  - IPv4 CIDR block: `10.0.1.0/24`
- Click **Create subnet**

### Step 2: Create Subnet 1b
- Click **Create subnet** again
- Configure:
  - VPC ID: Select `network-vpc`
  - Subnet name: `public-subnet-1b`
  - Availability Zone: `us-east-1b`
  - IPv4 CIDR block: `10.0.2.0/24`
- Click **Create subnet**

### Step 3: Enable Auto-assign Public IPs
- Select `public-subnet-1a` > Actions > Edit subnet settings
- Check **Enable auto-assign public IPv4 address** > Save
- Repeat for `public-subnet-1b`

---

## Part 3: Internet Connectivity

**Goal:** Enable internet connectivity for your subnets.

### Step 1: Create Internet Gateway
- Navigate to Internet Gateways > Create internet gateway
- Name: `network-igw`
- Click **Create internet gateway**
- Select the gateway > Actions > Attach to VPC > Select `network-vpc`

### Step 2: Create Route Table
- Go to Route Tables > Create route table
- Configure:
  - Name: `public-rt`
  - VPC: Select `network-vpc`
- Click **Create route table**

### Step 3: Add Route to Internet Gateway
- Select `public-rt` > Routes tab > Edit routes
- Click **Add route**
- Configure:
  - Destination: `0.0.0.0/0`
  - Target: Select Internet Gateway > `network-igw`
- Click **Save changes**

### Step 4: Associate Subnets
- Go to Subnet associations tab > Edit subnet associations
- Select both `public-subnet-1a` and `public-subnet-1b`
- Click **Save associations**

### What You Learned
- **VPC Creation** – Established your own isolated network with 65,536 IP addresses
- **Multi-AZ Subnets** – Created public subnets in two different Availability Zones for high availability
- **Internet Connectivity** – Configured an Internet Gateway and route tables to enable internet communication

**Next Lab:** Lab 2: EC2 – Launch compute instances in your new VPC.

---

## Part 4: EC2 (Elastic Compute Cloud)

EC2 provides virtual servers (instances) that you can launch on-demand. You pay only for what you use.

**Security Groups:** Think of Security Groups as firewalls for your instances. They control which traffic is allowed in (inbound) and out (outbound).

**User Data Scripts:** When you launch an instance, you can provide a script that runs automatically. This is perfect for installing software, starting services, or configuring the instance.

### What You'll Build
- Task 1: Create a Security Group allowing HTTP traffic
- Task 2: Launch an EC2 instance with a web server
- Task 3: Access the web server via HTTP using the public IP address

---

### Task 1: Create the Security Group

**Goal:** Create a firewall that allows HTTP traffic to your EC2 instance.

**Step 1: Create Security Group**
- Navigate to EC2 Dashboard > Security Groups
- Click **Create security group**
- Configure:
  - Security group name: `web-sg`
  - Description: Allow HTTP traffic for web servers
  - VPC: Select `network-vpc` *(corrected — original notes said "default VPC," which would orphan the VPC built in Part 1-3)*

**Step 2: Add Inbound Rules**
- Under Inbound rules, click **Add rule**
- Configure:
  - Type: HTTP
  - Protocol: TCP
  - Port range: 80
  - Source: `0.0.0.0/0` (Allow from anywhere)
- Click **Create security group**

**Verify:** Security group `web-sg` is created with HTTP inbound rule.

---

### Task 2: Launch EC2 Instance with Web Server

**Goal:** Launch an EC2 instance with a web server running.

**Step 1: Launch Instance**
- Navigate to EC2 Dashboard > Instances > Launch instances
- Configure:
  - Name: `web-server`
  - AMI: Amazon Linux 2023
  - Instance type: `t3.micro`
  - Key pair: Proceed without a key pair

**Step 2: Network Settings**
- Click **Edit** under Network settings
- Configure:
  - VPC: Select `network-vpc` *(corrected — original notes said "default VPC," which would orphan the VPC built in Part 1-3)*
  - Subnet: Select `public-subnet-1a`
  - Auto-assign public IP: Enable
  - Security group: Select `web-sg`

**Step 3: User Data Script**
- Expand Advanced details
- Scroll to User data and paste:

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo '<html><body style="background-color:blue; color:white"><h1>Hello from EC2!</h1></body></html>' > /var/www/html/index.html
```

- Click **Launch instance**

**Verify:** Instance `web-server` is running and has a public IP address.

---

### Task 3: Access the Web Server

**Goal:** Access your web server using the public IP address.

**Step 1: Get the Public IP Address**
- Navigate to EC2 Dashboard > Instances
- Select the `web-server` instance
- In the Details tab, find the Public IPv4 address (e.g., `54.123.45.67`)

**Step 2: Access the Web Server**
- Open a new browser tab
- Paste: `http://<public-ip-address>` (replace with your actual IP)
- You should see: **Hello from EC2!** on a blue background

**Troubleshooting:**
- If the page doesn't load, wait 1-2 minutes for the instance to fully boot and Apache to start
- Ensure the security group allows HTTP on port 80
- Verify the instance status is Running

---

## Part 5: Application Load Balancer (Pre-provisioned Resources)

**Goal:** Create an Application Load Balancer using pre-provisioned resources. The ALB will be the public entry point for your application.

**Step 1:** Go to EC2 → Load Balancers → Create load balancer.

**Step 2:** Choose Application Load Balancer.

**Step 3: Configure Basic Settings**
- Name: `alb-web-alb`
- Scheme: Internet-facing

**Step 4: Network Mapping**
- Select the default VPC
- Select all the availability zones

**Step 5: Security Group**
- Select the pre-created security group: `alb-web-sg`
- (Remove the default security group if selected)

**Step 6: Listeners and Routing**
- Protocol: HTTP
- Port: 80
- Default action: Forward to `alb-web-tg`

**Step 7:** Click **Create Load Balancer**.

---

## Part 6: Verify Target Group Health

**Goal:** Confirm the Target Group is healthy and ready to receive traffic. The ALB only sends traffic to targets that pass health checks.

**Step 1:** Go to EC2 → Target Groups.

**Step 2:** Click on `alb-web-tg`.

**Step 3:** Check the Targets tab:
- You should see both `alb-web-server-1` and `alb-web-server-2` registered
- The Health status should show Healthy (green) for both

**If status shows "Unhealthy" or "Initial":**
- Wait 1-2 minutes for health checks to complete
- Ensure the EC2 instances are running
- Verify the security group allows HTTP on port 80

---

## Part 7: Access the Application Through the ALB

**Goal:** Access your web application through the ALB. This proves the entire flow is working: ALB → Target Group → EC2.

**Step 1:** Go to EC2 → Load Balancers → `alb-web-alb`.

**Step 2:** In the Description tab, find the DNS name.
- It will look like: `alb-web-alb-123456789.us-east-1.elb.amazonaws.com`

**Step 3:** Copy the DNS name and paste it in a new browser tab (Add `http://` at the beginning).

**Step 4:** You should see either:
- Server 1 on a blue background
- Server 2 on a green background

**Step 5:** Refresh the page multiple times (5-10 times).
- You should see both servers, proving the ALB is distributing traffic

**Troubleshooting:**
- If you see an error, wait 1-2 minutes for the targets to become healthy
- Ensure the ALB status is Active (not Provisioning)

**Example DNS:**
```
http://alb-web-alb-1103584429.us-east-1.elb.amazonaws.com/
```

---

# Blue/Green Deployment Project

## Project Overview

In modern cloud environments, deploying new versions of an application shouldn't mean taking your system offline. This project builds a highly resilient infrastructure by deploying two identical, fully isolated environments (Blue and Green) behind an AWS Application Load Balancer (ALB).

You will learn how to transition traffic seamlessly from your live environment (Version 1 / Blue) to your updated environment (Version 2 / Green) using a Canary deployment strategy. Rather than flipping a switch and hoping for the best, a Canary strategy allows you to route a small percentage of user traffic (e.g., 20%) to the new version to monitor for errors before committing to a full 100% cutover. This ensures zero downtime and minimizes the blast radius if an update fails.

---

## Step 0: Create the Blue/Green Foundation Network (BG-VPC)

> **Note:** This section was added to fill a gap in the original notes — the Blue/Green project references `BG-VPC`, `Public-1`, and `Public-2` but never showed how they were created. This mirrors the same pattern used in Part 1-3 above, with a separate CIDR range so it doesn't collide with `network-vpc`.

**Goal:** Create the foundation network for the Blue/Green project.

### Step 1: Create the VPC
- Navigate to VPC Dashboard > Create VPC
- Configure:
  - Name tag: `BG-VPC`
  - IPv4 CIDR block: `10.1.0.0/16`
- Click **Create VPC**

### Step 2: Create Subnet Public-1
- Navigate to Subnets > Create subnet
- Configure:
  - VPC ID: Select `BG-VPC`
  - Subnet name: `Public-1`
  - Availability Zone: `us-east-1a`
  - IPv4 CIDR block: `10.1.1.0/24`
- Click **Create subnet**

### Step 3: Create Subnet Public-2
- Click **Create subnet** again
- Configure:
  - VPC ID: Select `BG-VPC`
  - Subnet name: `Public-2`
  - Availability Zone: `us-east-1b`
  - IPv4 CIDR block: `10.1.2.0/24`
- Click **Create subnet**

### Step 4: Enable Auto-assign Public IPs
- Select `Public-1` > Actions > Edit subnet settings
- Check **Enable auto-assign public IPv4 address** > Save
- Repeat for `Public-2`

### Step 5: Create and Attach Internet Gateway
- Navigate to Internet Gateways > Create internet gateway
- Name: `BG-igw`
- Click **Create internet gateway**
- Select the gateway > Actions > Attach to VPC > Select `BG-VPC`

### Step 6: Create Route Table
- Go to Route Tables > Create route table
- Configure:
  - Name: `BG-public-rt`
  - VPC: Select `BG-VPC`
- Click **Create route table**

### Step 7: Add Route to Internet Gateway
- Select `BG-public-rt` > Routes tab > Edit routes
- Click **Add route**
- Configure:
  - Destination: `0.0.0.0/0`
  - Target: Select Internet Gateway > `BG-igw`
- Click **Save changes**

### Step 8: Associate Subnets
- Go to Subnet associations tab > Edit subnet associations
- Select both `Public-1` and `Public-2`
- Click **Save associations**

**Verify:** `BG-VPC` exists with two public subnets (`Public-1` in `us-east-1a`, `Public-2` in `us-east-1b`), both with auto-assign public IP enabled and routed to the internet via `BG-igw`.

---

## Step 1: Launch the Blue Instance

We will launch our first server, representing the current "live" version of our application.

**Tasks:**

**Step 1: Launch Blue Instance**
- Dashboard: EC2 > Instances > Launch instances
- Name: `Blue-Server`
- AMI: Amazon Linux 2023
- Instance Type: `t3.micro`
- Key Pair: Proceed without a key pair (not needed for this lab).
- Network Settings: Click **Edit**.
  - VPC: `BG-VPC`
  - Subnet: `Public-1`
  - Security Group: Create security group > Name: `Web-SG` > Add rule: Allow HTTP (80) from Anywhere (`0.0.0.0/0`).
- Advanced Details (User Data): Scroll to the bottom and paste the following:

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo '<html><body style="background-color:blue; color:white"><h1>🟦 VERSION 1 (BLUE)</h1></body></html>' > /var/www/html/index.html
```

- Click: **Launch instance**

**Note:**
- Is the instance placed in `Public-1`?
- Does `Web-SG` allow inbound traffic on port 80?

---

## Step 2: Launch the Green Instance

**Tasks:**

**Step 1: Launch Green Instance**
- Dashboard: EC2 > Instances > Launch instances
- Name: `Green-Server`
- AMI: Amazon Linux 2023
- Instance Type: `t3.micro`
- Key Pair: Proceed without a key pair
- Network Settings: Click **Edit**.
  - VPC: `BG-VPC`
  - Subnet: `Public-2`
  - Security Group: Choose **Select existing security group** > Choose `Web-SG`. (Note: Uncheck the default VPC security group if it is selected).
- Advanced Details (User Data): Scroll to the bottom and paste the following:

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo '<html><body style="background-color:green; color:white"><h1>🟩 VERSION 2 (GREEN)</h1></body></html>' > /var/www/html/index.html
```

- Click: **Launch instance**

**Note:**
- Is the instance placed in `Public-2`?
- Did you reuse the `Web-SG` security group?

---

## Step 3: Create Target Groups

**Tasks:**

**Step 1: Create Target Group (Blue)**
- Dashboard: EC2 > Target Groups > Create target group
- Type: Instances
- Name: `TG-Blue`
- VPC: `BG-VPC`
- Protocol: HTTP (80) > Click **Next**.
- Register Targets: Select `Blue-Server` > Click **Include as pending below**.
- Click: **Create target group**

**Step 2: Create Target Group (Green)**
- Repeat the exact steps above, but name it `TG-Green` and register the `Green-Server`.

---

## Step 4: Create the Load Balancer

The Load Balancer will act as the single entry point for our users, allowing us to seamlessly shift traffic between the backend target groups.

**Tasks:**

**Step 1: Create Load Balancer**
- Dashboard: EC2 > Load Balancers > Create load balancer
- Type: Application Load Balancer (ALB) > Click **Create**.
- Name: `BlueGreen-ALB`
- Network mapping: Select `BG-VPC` | Check both `us-east-1a` (Public-1) and `us-east-1b` (Public-2).
- Security Groups: Deselect the default, and select `Web-SG`.
- Listeners and routing: Protocol HTTP (80) -> Default action: Forward to `TG-Blue`.
- Click: **Create load balancer**
- Note: Wait a few minutes for the Load Balancer status to become Active before proceeding.

**Note:**
- Is the Load Balancer routing to `TG-Blue` by default?
- Is the Load Balancer in the Active state?

---

## Step 5: Canary Deployment (Traffic Shifting)

Currently, 100% of traffic goes to the Blue instance. Let's shift it using a Canary deployment strategy.

**Tasks:**

**Step 1: Verify Status Quo**
- Copy the DNS Name of your ALB and paste it into a browser tab (ensure it starts with `http://`).
- Verify: You should see 🟦 VERSION 1 (BLUE).
- Refresh multiple times. It should stay Blue.

**Step 2: Modify Listener Rule (Canary Deployment)**
- Go to your Load Balancer > Listeners and rules tab.
- Select the Listener (HTTP:80) > Click **Manage rules** > Edit rule (Pencil icon).
- Under Routing Actions, change the weights:
  - Target Group 1: `TG-Blue` | Weight: 80
  - Target Group 2: `TG-Green` | Weight: 20
- Click: **Save changes**

**Step 3: WAIT — The Propagation Pause**
- Wait 30 to 60 seconds for the rules to update.

**Step 4: Test the Shift**
- Refresh your browser repeatedly.
- Result: Roughly 80% of the time you see BLUE, and 20% of the time you see GREEN.

**Step 5: Full Cutover**
- Edit the Rule again.
- Weights: `TG-Blue`: 0, `TG-Green`: 100.
- Click: **Save changes**.
- Result: 100% of users now see 🟩 VERSION 2 (GREEN).

**Note:**
- Were you able to successfully see the 80/20 traffic split?
- After the full cutover, does the ALB successfully route 100% of traffic to the Green environment?

---

## Recap: Skills Mastered

You have successfully executed an enterprise-grade deployment strategy in AWS!

By utilizing a Blue/Green architecture combined with a Canary traffic shift, you have learned how to safely introduce new application versions to a subset of users. This methodology minimizes downtime, drastically reduces risk, and ensures a seamless experience for your end-users.

Here is a recap of the enterprise-grade skills you just mastered:

- **Advanced VPC Networking:** Provisioning multi-AZ subnets to ensure high availability and fault tolerance for load balancers.
- **EC2 Provisioning:** Automating web server configuration using User Data scripts upon boot.
- **Target Group Decoupling:** Isolating distinct application environments into separate backend target groups for granular traffic control.
- **Application Load Balancers (ALB):** Configuring listener rules to act as the central routing mechanism for your users.
- **Canary Deployments:** Shifting precise percentages of live traffic between active environments to safely test new releases in production.

Thank you for learning with KodeKloud! Keep building!
