# Migrating an EC2 Instance's Private + Public IP to a New Instance Using ENI & Elastic IP
### Complete Hands-On Lab Guide (Console-First)

**Author:** Saime Shaikh
**Level:** Hands-on / Interview-ready
**Scope:** AWS EC2, VPC, ENI (Elastic Network Interface)

---

## 1. Objective

You have **EC2-A** (old instance) with a **private IP** and an **auto-assigned public IP** (no Elastic IP originally attached). You want to launch **EC2-B** (new instance) and have it **inherit both the private IP (via ENI) and a stable public IP (via Elastic IP)**, without CLI in the lab steps.

This lab has two independent tracks that both feed into the same end state:
- **Track 1 — Private IP**: moved via ENI surgery (Sections 5–6)
- **Track 2 — Public IP**: moved via Elastic IP allocation/association (Section 7) — this is a *separate* AWS mechanism, not an ENI termination-behavior trick, because auto-assigned public IPs cannot be preserved that way (see Section 2.4)

This is a real production pattern used for:
- Instance replacement / patching without changing IP-dependent firewall rules, DNS records, or hardcoded configs
- Blue-green style cutover at the network layer
- Disaster recovery / failover setups
- Avoiding Elastic IP costs while still getting "IP portability"

---

## 2. Core Concepts You MUST Understand Before Starting

### 2.1 What is an ENI?

An **Elastic Network Interface (ENI)** is a virtual network card in a VPC. It holds:
- One primary private IPv4 address
- Optional secondary private IPv4 addresses
- One or more security groups
- A MAC address
- An Elastic IP (optional, associated per private IP)
- A source/destination check flag

Every EC2 instance has **at least one ENI** — the **primary network interface**, created automatically at launch time (`eth0`), in the subnet you chose.

### 2.2 The Critical Limitation (This is the whole trick of this lab)

> **You CANNOT detach the primary network interface (eth0) from a running instance.**

AWS explicitly blocks this. This means:

| Scenario | Can you just "unplug and replug" the ENI while both instances are alive? |
|---|---|
| The IP lives on a **secondary ENI** attached to EC2-A | ✅ Yes — hot detach/attach, near-zero downtime |
| The IP lives on the **primary ENI (eth0)** of EC2-A (this is the default/normal case for a simple instance) | ❌ No — you must **terminate EC2-A first** (with `DeleteOnTermination=No` set on that ENI), which frees the ENI, then attach it to EC2-B as a **secondary** interface |

Most people asking "move my EC2's IP to a new instance" are in the **second case**, because when you launch a normal EC2 instance, AWS auto-creates a primary ENI and by default sets **"Delete on Termination = Yes"** for it. If you don't flip that flag, the ENI (and the private IP) dies with the instance — permanently gone.

So this lab covers **both paths**:

- **Lab Path A** — Live migration (secondary ENI already in use) — ideal, no downtime, repeatable.
- **Lab Path B** — Primary ENI recovery migration (the "IP is on eth0" case) — has brief downtime, but works for your exact scenario (existing EC2 with a normal, non-elastic private IP).

Since your situation is "1 EC2 with some IP, not Elastic IP" — this is almost certainly a **primary ENI** situation. **Path B is your real task.** Path A is included because it's the better long-term design pattern going forward (build it right the second time).

### 2.3 Private IP vs Elastic IP — Why This Matters Here

| | Private IP | Elastic IP |
|---|---|---|
| Scope | Internal to VPC | Public, internet-routable |
| Tied to | The ENI (not the instance) | Also tied to an ENI, but can be remapped instantly via API |
| Cost | Free | Free while attached to running instance; charged if idle/unattached |
| Portability | Only portable if the ENI itself is portable | Can be re-associated to any ENI in seconds |
| Your case | You're moving this | Not used in your scenario |

An Elastic IP is easy to "move" (just re-associate it). A **plain private IP is only movable if you move the ENI it lives on** — that's why ENI surgery is required here.

### 2.4 Public IP Behavior — Why "Delete on Termination" Doesn't Help Here

This is the part most guides skip, and it directly affects your task:

| IP Type | Where it actually lives | Survives instance termination? | How to preserve it for a new instance |
|---|---|---|---|
| Private IP | On the ENI itself | Yes — if the ENI's `Delete on Termination` is set to `No` | Detach/reattach the ENI (Sections 5–6) |
| Auto-assigned Public IP | Dynamically mapped by AWS to the ENI's primary private IP, only while the instance is running | **No — never.** Released back to AWS's pool the moment the instance stops or terminates, regardless of any ENI setting | Not possible — there is no "preserve" option for this IP type |
| Elastic IP | A separately-allocated, account-owned address, associated to a private IP on an ENI | Yes — persists independently of any instance's lifecycle | Disassociate from the old instance/ENI, associate to the new one (Section 7) |

**Bottom line:** ENI termination-behavior settings only ever protect the *private* IP. If you also need a stable *public* IP across instance replacement, you must use an Elastic IP — there is no ENI-only path to preserving a plain auto-assigned public IP.

### 2.5 Same AZ Requirement

**An ENI can only be attached to an instance in the same Availability Zone.** It is NOT region-wide, it's not even VPC-wide — it's AZ-locked at creation time. EC2-B **must** be launched in the same AZ (and same subnet, since the ENI's private IP belongs to a specific subnet's CIDR range) as EC2-A.

---

## 3. Architecture Diagrams

### 3.1 Before Migration (Current State)

```
                         VPC: 10.0.0.0/16
                         AZ: ap-south-1a
                         Subnet: 10.0.1.0/24
        ┌───────────────────────────────────────────┐
        │                                             │
        │   ┌─────────────────────────┐               │
        │   │   EC2-A  (running)      │               │
        │   │                         │               │
        │   │   eth0 (Primary ENI)    │               │
        │   │   IP: 10.0.1.25         │               │
        │   │   DeleteOnTermination:  │               │
        │   │        YES (default) ⚠  │               │
        │   └─────────────────────────┘               │
        │                                             │
        └───────────────────────────────────────────┘
```

### 3.2 Step 1 — Flip the flag, then Terminate EC2-A

```
        ┌───────────────────────────────────────────┐
        │                                             │
        │   ┌─────────────────────────┐               │
        │   │   EC2-A  (terminated)   │               │
        │   │   [instance destroyed]  │               │
        │   └─────────────────────────┘               │
        │                                             │
        │   ┌─────────────────────────┐               │
        │   │  ENI (orphaned/free)     │              │
        │   │  IP: 10.0.1.25          │               │
        │   │  Status: available       │              │
        │   └─────────────────────────┘               │
        │      (Survived because we set               │
        │       DeleteOnTermination = No)              │
        │                                             │
        └───────────────────────────────────────────┘
```

### 3.3 Step 2 — Launch EC2-B, Attach the Old ENI as Secondary

```
        ┌───────────────────────────────────────────┐
        │                                             │
        │   ┌─────────────────────────────────────┐   │
        │   │   EC2-B  (new instance)              │   │
        │   │                                       │   │
        │   │   eth0 (new Primary ENI)              │   │
        │   │   IP: 10.0.1.90  (new, auto-assigned) │   │
        │   │                                       │   │
        │   │   eth1 (attached secondary ENI)       │   │
        │   │   IP: 10.0.1.25  ← the migrated IP    │   │
        │   └─────────────────────────────────────┘   │
        │                                             │
        └───────────────────────────────────────────┘
```

> ⚠️ Note: EC2-B ends up with **two IPs** — its own new primary IP, plus the old migrated IP on `eth1`. If anything (DNS, firewall rules, clients) was pointing to `10.0.1.25`, it will now reach EC2-B correctly via that secondary interface — but only after you configure the OS routing (Section 6), otherwise the OS will silently drop return traffic on eth1.

---

## 4. Prerequisites

- An AWS account with an existing running EC2 instance (**EC2-A**) that has a private IP you want to preserve
- IAM permissions: `ec2:DescribeInstances`, `ec2:DescribeNetworkInterfaces`, `ec2:ModifyNetworkInterfaceAttribute`, `ec2:DetachNetworkInterface`, `ec2:AttachNetworkInterface`, `ec2:TerminateInstances`, `ec2:RunInstances`
- Know the AZ and Subnet ID of EC2-A (you'll need to launch EC2-B in the exact same one)
- SSH key pair access to both instances for verification
- 15–20 minutes, and accept that there **will be a short downtime window** for Path B (the instance is literally gone for a few minutes while the IP has no compute behind it)

---

## 5. LAB PATH A — Live Migration (Secondary ENI, No Downtime)

Use this if you want to **design this correctly going forward** — attach a dedicated secondary ENI to any critical instance from day one, so future migrations need zero downtime.

### Step A1 — Create a Standalone ENI

1. Go to **VPC Console → Network Interfaces → Create network interface**
2. **Description**: `app-floating-eni`
3. **Subnet**: choose the same subnet as EC2-A
4. **Private IPv4 address**: choose **Custom** and manually type the private IP you want to "own" going forward (or Auto-assign if this is a fresh setup)
5. **Security groups**: attach the same security group(s) as EC2-A
6. Click **Create network interface**

### Step A2 — Attach This ENI to EC2-A as a Secondary Interface

1. Select the new ENI → **Actions → Attach**
2. Choose **EC2-A** as the instance
3. Confirm attachment — it becomes `eth1` on EC2-A

### Step A3 — Configure the OS on EC2-A to Use eth1

SSH into EC2-A and run (this is OS-side, not an AWS CLI call against the API, so it's still "hands-on lab" work, not the automation you excluded):

```bash
# See the new interface
ip addr show

# Bring it up if not already
sudo dhclient eth1
```

> On Amazon Linux 2/2023, secondary ENIs are usually auto-configured on boot via **ec2-net-utils**. If traffic isn't returning correctly, jump to Section 6 (policy-based routing) — the same fix applies here.

### Step A4 — The Actual "Migration" Moment

When you're ready to cut over to EC2-B:

1. **VPC Console → Network Interfaces** → select `app-floating-eni`
2. **Actions → Detach** → confirm (this is a **live** detach, works because it's NOT the primary ENI of EC2-A)
3. Once state shows **available**, select it again → **Actions → Attach**
4. Choose **EC2-B** as the target instance
5. It appears as `eth1` on EC2-B automatically
6. Repeat the OS-level config from Step A3 on EC2-B

**Total downtime with this path: seconds** (just the detach→attach window), and EC2-A can keep running the whole time on its own primary IP.

---

## 6. LAB PATH B — Your Actual Scenario (Primary ENI → New Instance)

This is the path for **"I have 1 EC2 with a normal private IP, no Elastic IP, and I want a new EC2 to take over that exact IP."**

### Step B1 — Identify Everything About EC2-A First

1. Go to **EC2 Console → Instances** → select EC2-A
2. Note down from the **Details** tab:
   - **Private IPv4 address** (e.g., `10.0.1.25`) ← this is what you're preserving
   - **VPC ID**
   - **Subnet ID**
   - **Availability Zone**
   - **Security group(s)**
3. Go to the **Networking** tab → note the **Network Interface ID** (e.g., `eni-0abc123...`) — this is EC2-A's primary ENI

### Step B2 — Disable "Delete on Termination" for This ENI (CRITICAL STEP)

If you skip this, the ENI and IP are destroyed the moment you terminate EC2-A — permanently, no recovery.

1. Still on the **Networking** tab of EC2-A, click on the **Network Interface ID**
2. This opens the ENI details in the Network Interfaces console
3. Select the ENI → **Actions → Change termination behavior**
4. Set **Delete on termination** to **No**
5. Save

> Verify this actually saved by refreshing and re-checking — this is the single most common mistake in this entire lab. If this flag is still "Yes" when you terminate, everything below is unrecoverable and you'll need to reconfigure a fresh IP from scratch.

### Step B3 — Terminate EC2-A

1. **EC2 Console → Instances** → select EC2-A
2. **Instance state → Terminate instance**
3. Confirm
4. Wait until the instance state shows **Terminated**

### Step B4 — Verify the ENI Survived

1. **VPC Console → Network Interfaces**
2. Search for the ENI ID you noted in Step B1
3. Its **Status** should now show **Available** (not "in-use", not gone)
4. Confirm the **Private IPv4 address** field still shows `10.0.1.25`

If you don't see it at all, the Delete on Termination flag wasn't actually applied — this only works if Step B2 was done correctly *before* termination.

### Step B5 — Launch EC2-B (the New Instance)

1. **EC2 Console → Launch Instance**
2. Name: `EC2-B`
3. Choose AMI, instance type as needed
4. **Network settings → Edit**:
   - **VPC**: same VPC as EC2-A
   - **Subnet**: the **exact same subnet** EC2-A was in (mandatory — ENI is subnet-bound)
   - Leave **Auto-assign public IP** as needed for your use case
   - Security group: same or equivalent to EC2-A's
5. Launch the instance and wait until it's in **Running** state

> Do NOT try to manually set EC2-B's primary IP to `10.0.1.25` at launch — that IP belongs to a detached ENI, not a fresh one, and launch-time IP assignment creates a brand-new primary ENI. We're going to attach the *old* ENI as a *secondary* interface instead.

### Step B6 — Attach the Old ENI to EC2-B

1. **VPC Console → Network Interfaces**
2. Select the old ENI (still showing IP `10.0.1.25`, status: Available)
3. **Actions → Attach network interface**
4. Choose **EC2-B** as the instance
5. Confirm — AWS assigns it as `eth1` (device index 1) on EC2-B

### Step B7 — (Optional but recommended) Re-enable Delete on Termination for EC2-B

Now that the ENI is attached to its new permanent home:

1. Select the ENI → **Actions → Change termination behavior**
2. Set **Delete on termination** back to **Yes**, associated with EC2-B

This prevents an orphaned ENI next time you terminate this instance, and avoids surprise leftover ENI charges/clutter.

---

## 7. Preserving the Public IP with an Elastic IP (Required — Auto-Assigned Public IPs Cannot Survive Termination)

### 7.1 Why This Section Exists

The ENI trick in Section 6 preserves the **private IP** because the private IP genuinely lives on the ENI. A **public IP that AWS auto-assigns** at launch does **not** work the same way — it is only mapped to the ENI's primary private IP for as long as the instance is running. The instant you stop or terminate the instance, that public IP is released back into AWS's general pool permanently. Setting `Delete on Termination = No` on the ENI has **zero effect** on this — it only protects the private IP side.

There is exactly one AWS mechanism that gives you a public IP that survives instance replacement: an **Elastic IP (EIP)**. This section adds that piece so both the private IP and the public IP land on EC2-B.

### 7.2 Important Sequencing Note

Do this **before** you terminate EC2-A (Step B3), while EC2-A is still running and reachable — that way you have zero time where nobody has that public IP, and you can confirm it worked without needing to first tear anything down. It does not depend on the ENI/private-IP steps at all — these two procedures are independent of each other and can be done in either order.

### 7.3 Step E1 — Allocate a New Elastic IP

1. **EC2 Console → Network & Security → Elastic IPs**
2. Click **Allocate Elastic IP address**
3. **Network Border Group**: leave as default (matches your region)
4. **Public IPv4 address pool**: choose **Amazon's pool of IPv4 addresses** (unless you specifically own BYOIP addresses)
5. Click **Allocate**
6. Note the new address, e.g. `52.66.123.45`

### 7.4 Step E2 — Associate the Elastic IP to EC2-A (Optional Validation Step)

This step is optional — it just proves the EIP behaves the way you expect before you rely on it for the real cutover. Skip straight to Step E3 if you'd rather not touch EC2-A at all.

1. Select the newly allocated EIP → **Actions → Associate Elastic IP address**
2. **Resource type**: Instance
3. **Instance**: EC2-A
4. **Private IP address**: select EC2-A's private IP (the same `10.0.1.25` from Section 5)
5. Click **Associate**
6. Confirm EC2-A now shows this EIP as its public IPv4 address in the instance details

> Note: associating an EIP to EC2-A does **not** remove or replace its existing auto-assigned public IP slot permanently — it simply overrides what's shown as the instance's public-facing address. The old auto-assigned public IP (if any) is dropped once an EIP is associated.

### 7.5 Step E3 — Disassociate the Elastic IP from EC2-A

Before terminating EC2-A, release the association (this does **not** delete the EIP — it just frees it up to be reattached elsewhere):

1. **Elastic IPs** console → select the EIP → **Actions → Disassociate Elastic IP address**
2. Confirm
3. The EIP now shows **no associated instance**, and stays fully reserved to your account

> If you skip this and terminate EC2-A directly, AWS automatically disassociates the EIP anyway (EIPs are never deleted on instance termination — only auto-assigned public IPs are). Doing it manually first is just cleaner and lets you verify state at each step.

### 7.6 Step E4 — Associate the Elastic IP to EC2-B

Do this any time after EC2-B is up and running (Step B5 in Section 6):

1. **Elastic IPs** console → select the EIP → **Actions → Associate Elastic IP address**
2. **Resource type**: Instance
3. **Instance**: EC2-B
4. **Private IP address**: choose the private IP on EC2-B that should be internet-reachable
   - If you want the public IP to route to the **migrated private IP** (`10.0.1.25` on `eth1`), select that one
   - If you want it to route to EC2-B's own **new native private IP** (`eth0`), select that instead
5. Click **Associate**

> This choice matters: an EIP is associated to exactly one private IP on one ENI. Pick whichever private IP is meant to be the "front door" for this instance going forward.

### 7.7 Verify

```bash
curl https://checkip.amazonaws.com
```

Run this from your own machine against EC2-B's public address (or just check the EC2 console — the **Public IPv4 address** field for EC2-B should now show the EIP you allocated, e.g. `52.66.123.45`).

### 7.8 Cost Note

- An Elastic IP is **free** while it's associated with a running instance.
- It is **billed hourly** the moment it's unassociated (sitting idle) or associated with a stopped instance.
- Since Step E3 leaves it briefly unassociated, this is normal and the cost impact is negligible (a few cents at most) as long as you complete Step E4 promptly.

---

## 8. OS-Level Configuration on EC2-B (Mandatory — Don't Skip)

At this point, AWS has done its job — the IP is attached to EC2-B. But **Linux will not automatically route traffic correctly across two interfaces on two different subnets/IPs** without help. This step trips up almost everyone.

### 7.1 SSH Into EC2-B (via its NEW primary IP, not the migrated one — it's not routable yet)

```bash
ssh -i your-key.pem ec2-user@<EC2-B-primary-ip>
```

### 7.2 Confirm the Interface Is Visible

```bash
ip addr show
```

You should see `eth0` (primary, new IP) and `eth1` (secondary, migrated IP `10.0.1.25`) — but `eth1` may show **no IP assigned** yet, because the OS doesn't auto-configure secondary ENIs unless told to.

### 7.3 Amazon Linux 2 / 2023 — Use ec2-net-utils (Usually Pre-Installed)

```bash
# Check if it's installed
rpm -qa | grep ec2-net-utils

# If missing, install it
sudo yum install -y ec2-net-utils

# Restart networking to pick up the new ENI
sudo systemctl restart NetworkManager
```

Amazon Linux's ec2-net-utils automatically detects the new ENI, assigns the IP, and — critically — sets up the necessary **policy-based routing rules** (separate routing tables per interface) so return traffic doesn't get dropped.

### 7.4 Manual Configuration (Ubuntu / any distro without ec2-net-utils)

If auto-config doesn't kick in, do this manually. This is the real meat of "why does my second ENI not work" issues:

```bash
# 1. Bring the interface up and assign the IP manually
sudo ip addr add 10.0.1.25/24 dev eth1
sudo ip link set eth1 up

# 2. Create a separate routing table for eth1 (e.g., table 1001)
echo "1001 eth1-rt" | sudo tee -a /etc/iproute2/rt_tables

# 3. Add a default route via eth1's subnet gateway, in that table
sudo ip route add default via 10.0.1.1 dev eth1 table eth1-rt

# 4. Add a rule: traffic FROM 10.0.1.25 uses the eth1 table
sudo ip rule add from 10.0.1.25 lookup eth1-rt

# 5. Verify
ip rule show
ip route show table eth1-rt
```

> Replace `10.0.1.1` with the actual gateway of that subnet (normally the subnet's network address +1, e.g. subnet `10.0.1.0/24` → gateway `10.0.1.1`).

**Why this is needed:** Without policy routing, the kernel picks the "shortest path" default route (usually `eth0`'s), so requests arriving on `eth1` get replies sent back out via `eth0` with a mismatched source IP — the client's OS or a stateful firewall/AWS VPC's source/dest check will silently drop it. This is the single most common "it's attached but not working" issue in the field.

### 7.5 Persist Config Across Reboot

Manual `ip` commands are cleared on reboot. Persist them:

**Ubuntu (netplan):**
```yaml
# /etc/netplan/60-eth1.yaml
network:
  version: 2
  ethernets:
    eth1:
      addresses: [10.0.1.25/24]
      routes:
        - to: 0.0.0.0/0
          via: 10.0.1.1
          table: 1001
      routing-policy:
        - from: 10.0.1.25
          table: 1001
```
```bash
sudo netplan apply
```

**RHEL/CentOS/Amazon Linux (network-scripts style):**
```bash
# /etc/sysconfig/network-scripts/ifcfg-eth1
DEVICE=eth1
BOOTPROTO=static
IPADDR=10.0.1.25
PREFIX=24
ONBOOT=yes
```

---

## 9. Verification Checklist

Run these before declaring success:

```bash
# 1. Interface has the IP
ip addr show eth1
# Expect: inet 10.0.1.25/24 ... on eth1

# 2. Routing rule exists
ip rule show
# Expect: a line like "from 10.0.1.25 lookup eth1-rt"

# 3. Can EC2-B reach out from that IP specifically?
curl --interface eth1 https://checkip.amazonaws.com

# 4. From another host in the VPC, ping the migrated IP
ping 10.0.1.25

# 5. Check AWS-side: does the console show the ENI as "in-use" and attached to EC2-B?
```

**AWS Console check:**
- VPC → Network Interfaces → the ENI shows **Status: In-use**, **Instance ID: EC2-B**, **Private IP: 10.0.1.25**

**Public IP / Elastic IP check:**
- EC2 Console → Instances → EC2-B → **Public IPv4 address** field shows your allocated Elastic IP
- `curl https://checkip.amazonaws.com` from outside AWS returns that same address
- Elastic IPs console → the EIP shows **Associated instance ID: EC2-B**, not "unassociated"

---

## 10. CLI Equivalent (Reference Only — Not Required for the Lab)

Since your lab preference is console-first, here's the CLI mapping purely as a reference/cheat sheet — useful for interviews or scripting later:

```bash
# Describe EC2-A's ENI
aws ec2 describe-instances --instance-ids i-xxxxxEC2A \
  --query "Reservations[].Instances[].NetworkInterfaces[]"

# Disable delete-on-termination on the ENI (attachment-id required)
aws ec2 modify-network-interface-attribute \
  --network-interface-id eni-0abc123 \
  --attachment AttachmentId=eni-attach-0xyz456,DeleteOnTermination=false

# Terminate EC2-A
aws ec2 terminate-instances --instance-ids i-xxxxxEC2A

# Confirm ENI is now "available"
aws ec2 describe-network-interfaces --network-interface-ids eni-0abc123

# Attach ENI to EC2-B
aws ec2 attach-network-interface \
  --network-interface-id eni-0abc123 \
  --instance-id i-xxxxxEC2B \
  --device-index 1

# (Optional) Re-enable delete-on-termination for the new attachment
aws ec2 modify-network-interface-attribute \
  --network-interface-id eni-0abc123 \
  --attachment AttachmentId=eni-attach-0new789,DeleteOnTermination=true
```

---

## 11. Terraform Reference

```hcl
resource "aws_network_interface" "floating_eni" {
  subnet_id       = aws_subnet.app_subnet.id
  private_ips     = ["10.0.1.25"]
  security_groups = [aws_security_group.app_sg.id]

  tags = {
    Name = "floating-app-eni"
  }
}

resource "aws_instance" "ec2_b" {
  ami           = "ami-xxxxxxxx"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.app_subnet.id

  # Primary ENI is auto-created by the instance resource itself
}

resource "aws_network_interface_attachment" "attach_to_ec2b" {
  instance_id          = aws_instance.ec2_b.id
  network_interface_id = aws_network_interface.floating_eni.id
  device_index         = 1
}

# Public IP track — independent of the ENI resources above
resource "aws_eip" "floating_eip" {
  domain = "vpc"

  tags = {
    Name = "floating-app-eip"
  }
}

resource "aws_eip_association" "eip_to_ec2b" {
  instance_id   = aws_instance.ec2_b.id
  allocation_id = aws_eip.floating_eip.id
  # Optionally pin to a specific private IP if the instance has more than one:
  # private_ip_address = aws_network_interface.floating_eni.private_ip
}
```

> In Terraform-managed infra, the cleanest pattern is to **always** define the "floating" ENI as its own resource (like above) from day one, and reference it independently of any specific instance — that's the infra equivalent of Lab Path A, and avoids ever needing Lab Path B's downtime dance again.

---

## 12. Edge Cases & Failure Scenarios

| Scenario | What Happens | Fix |
|---|---|---|
| Forgot to disable Delete on Termination before terminating EC2-A | ENI is deleted permanently along with the instance | No recovery — must assign a fresh IP; always double-check Step B2 before terminating anything |
| Launch EC2-B in a different subnet | ENI attach will fail — "different subnet" error | ENI is subnet-locked; EC2-B must be in the exact same subnet as the ENI |
| Launch EC2-B in a different AZ | Attach fails outright | ENI is AZ-locked; must match |
| Instance type doesn't support multiple ENIs | Attach fails — "maximum number of network interfaces reached" | Check ENI limits per instance type (e.g., `t3.nano` supports only 2 ENIs total incl. primary); pick a larger type if needed |
| Security group mismatch between eth0 and eth1 | Traffic on eth1 gets blocked even though attached correctly | Explicitly attach the correct SG to the ENI, independent of the instance's own default SG |
| Source/Destination check enabled + eth1 used for pass-through/NAT-like traffic | Traffic silently dropped | Disable **Source/Dest Check** on the ENI if it's meant to route traffic not addressed to itself (not needed for simple "own the IP" cases) |
| OS doesn't bring up eth1 automatically after reboot | IP "disappears" after every reboot | Persist config in netplan/ifcfg as shown in Section 7.5, don't rely on runtime `ip addr add` |
| No policy routing configured | ENI is "attached" in console but host is unreachable on that IP | This is the #1 real-world failure — always configure Section 7.4 explicitly, don't assume auto-detection works on non-Amazon-Linux AMIs |
| DNS or firewall rules pointing to the old IP expecting zero downtime | There WILL be downtime in Path B between termination and successful attach+config on EC2-B | If zero downtime is a hard requirement, redesign using Path A (dedicated floating ENI) before this ever needs to happen again |

---

## 13. Best Practices

- **Never rely on primary ENI IPs as "stable" identifiers** for anything long-lived (DNS, firewall allowlists, hardcoded IPs). Use a dedicated secondary ENI (Path A pattern) or an Elastic IP if internet-facing.
- Always tag ENIs clearly (`Name`, `Environment`, `Owner`) — an orphaned untagged ENI floating in "available" state is a common source of confusion and clutter in shared accounts.
- Set a **budget/cost alert** — unattached ENIs themselves are free, but if you also carry an unattached Elastic IP anywhere in the account, that's billed hourly.
- Document which ENI is "the floating one" in your infra-as-code / runbooks so the next engineer doesn't accidentally delete it thinking it's unused.
- Where possible, **prefer Path A design from the start** for anything that will need this kind of migration more than once — it's the difference between seconds of downtime and minutes.
- If this is for a public-facing service, strongly consider an **Elastic IP + Elastic IP re-association** instead of a private-IP-only ENI dance — EIP re-association is a single API call with near-zero downtime and no OS-level routing gymnastics.

---

## 14. Cleanup (After You're Done Practicing)

1. Terminate **EC2-B** (this also deletes the attached ENI if Delete on Termination = Yes, as set in Step B7)
2. If you created a standalone ENI in Lab Path A and it's still sitting unattached, manually delete it: **VPC → Network Interfaces → select → Actions → Delete**
3. Double check **VPC → Network Interfaces** for any leftover "available" ENIs from practice runs — these don't cost money but clutter the console
4. **Release the Elastic IP** if you're done practicing: Elastic IPs console → select it → **Actions → Release Elastic IP addresses**. An unassociated EIP left sitting in your account is billed hourly — this is the one item in this whole lab that actively costs money if forgotten

---

## 15. Interview Q&A

**Q1: Can you detach the primary network interface of a running EC2 instance?**
A: No. AWS does not allow detaching the primary ENI (device index 0) from a running instance under any circumstance. You can only detach secondary ENIs while the instance is running. To reclaim a primary ENI's IP, the instance must be terminated first (with Delete on Termination disabled on that ENI).

**Q2: What determines whether an ENI can be attached to a given instance?**
A: The ENI's Availability Zone and Subnet must match the target instance's. Attaching across AZs or subnets is not permitted.

**Q3: What happens to a network interface by default when its instance is terminated?**
A: By default, the primary ENI has `DeleteOnTermination = true`, meaning it's deleted along with the instance. Secondary ENIs default to `false` and survive termination unless explicitly changed.

**Q4: Why would traffic not return correctly on a secondary ENI even after it's successfully attached?**
A: Because Linux uses a single default routing table by default, which typically routes all outbound traffic via eth0. Without policy-based routing (separate routing table + `ip rule` for the secondary IP), replies to traffic received on eth1 get sent out via eth0 with a mismatched source, and get dropped.

**Q5: How many ENIs can you attach to an instance?**
A: It depends on instance type/size — smaller instance types support fewer ENIs (e.g., 2–3 total including primary), larger types support more. Always check the instance type's network interface limit in AWS documentation before planning multi-ENI architectures.

**Q6: What's the difference between moving an Elastic IP vs moving a private IP via ENI?**
A: An Elastic IP can be re-associated to any ENI/instance in the same region via a single API call/console action, in seconds, with no OS reconfiguration needed. A plain private IP has no such API — it's physically tied to whichever ENI it lives on, so "moving" it means physically detaching/attaching (or terminate/re-attach) that ENI, plus manual OS-level routing configuration on the receiving instance.

**Q7: Why disable Source/Destination Check on an ENI, and is it needed here?**
A: Source/Dest check ensures traffic is only accepted/sent if the instance is the actual source or destination — required for NAT instances, transparent proxies, or routing appliances. It is NOT needed for this lab's use case, since EC2-B legitimately owns the migrated IP as its own address.

**Q8: Can you preserve an auto-assigned public IP across instance termination using ENI settings, the same way you preserve a private IP?**
A: No. `Delete on Termination` on an ENI only protects the private IP that lives on that ENI. An auto-assigned public IP is a separate, dynamically-mapped address that AWS releases back to its pool the instant the instance stops or terminates — no ENI setting can prevent this. The only way to get a public IP that survives instance replacement is to use an Elastic IP, which is allocated and associated independently of any ENI's termination behavior.

**Q9: What's the actual mechanical difference between how a private IP and an Elastic IP get "moved" to a new instance?**
A: A private IP moves by physically relocating the ENI it lives on (detach from old instance/attach to new one, or terminate-then-attach if it was on a primary ENI). An Elastic IP moves without touching any ENI hardware at all — you simply disassociate it from wherever it currently points and associate it to a private IP on the new instance's ENI. The EIP itself never leaves your account; only its association target changes.

---

## 16. Quick Cheat Sheet

```
PRIVATE IP TRACK (via ENI)
IDENTIFY   → Note EC2-A's ENI ID, subnet, AZ, private IP, SGs
PROTECT    → Set ENI's Delete on Termination = No
DESTROY    → Terminate EC2-A
CONFIRM    → ENI status = "available", IP still shows correctly
LAUNCH     → EC2-B in SAME subnet + AZ
ATTACH     → Old ENI to EC2-B as secondary (eth1)
CONFIGURE  → OS: assign IP to eth1 + policy-based routing (Section 8)
VERIFY     → ip addr, ip rule, curl --interface, ping from another host
LOCK IN    → Set Delete on Termination = Yes on new attachment

PUBLIC IP TRACK (via Elastic IP — independent of ENI settings)
ALLOCATE   → Elastic IPs console → Allocate Elastic IP address
(optional) → Associate to EC2-A first to sanity-check behavior
DISASSOC   → Disassociate EIP from EC2-A before/after termination
ASSOCIATE  → Associate EIP to a private IP on EC2-B
VERIFY     → curl https://checkip.amazonaws.com from outside AWS
CLEANUP    → Release the EIP when done — this is the one thing billed hourly if left idle
```

---

## 17. Mastery Checklist

- [ ] I can explain why the primary ENI can't be detached from a running instance
- [ ] I know the difference between Path A (live, secondary ENI) and Path B (terminate + reattach, primary ENI)
- [ ] I successfully disabled Delete on Termination BEFORE terminating an instance
- [ ] I confirmed the ENI survived termination and shows "available"
- [ ] I attached the surviving ENI to a new instance in the same subnet/AZ
- [ ] I configured the OS (netplan or ifcfg) to bring up the secondary interface with the correct IP
- [ ] I configured policy-based routing (`ip rule` + separate routing table) so return traffic works
- [ ] I verified end-to-end connectivity to the migrated IP from another host
- [ ] I persisted the network config so it survives a reboot
- [ ] I understand when to prefer an Elastic IP over this entire ENI dance
- [ ] I can explain why ENI termination-behavior settings do NOT protect auto-assigned public IPs
- [ ] I successfully allocated an Elastic IP and associated it to the new instance
- [ ] I remembered to release the Elastic IP during cleanup to avoid idle charges
- [ ] I can articulate all of the edge cases in Section 12 without looking them up

---

**End of Guide.**
