# Cross-Account Transit Gateway Peering Lab 

---

## Table of Contents

1. Lab Overview & Objectives
2. Architecture Diagram (Overview)
3. Prerequisites
4. IP Addressing Plan
5. Part A — Account A: VPC, Subnet, IAM Role, SSM Endpoints, EC2
6. Part B — Account B: VPC, Subnet, IAM Role, SSM Endpoints, EC2
7. Part C — Create Transit Gateways (One per Account)
8. Part D — Attach Each VPC to Its Own TGW
9. Part E — Cross-Account TGW Peering Attachment
10. Part F — VPC Route Table Updates (Post-Peering)
11. Part G — TGW Route Table Updates (Static Routes)
12. Security Groups / NACL Considerations
13. Complete Packet Flow Walkthrough
14. Connecting via Session Manager & Ping Testing — Both Directions
15. Troubleshooting Checklist
16. VPC Peering vs. Transit Gateway Peering — Comparison
17. Final Architecture Diagram (Consolidated, With Route Paths)
18. Cleanup (Dependency-Ordered)

---

## 1. Lab Overview & Objectives

This lab connects **two AWS accounts**, each with one fully private VPC, using a **Transit Gateway Peering Attachment** between two independently owned Transit Gateways — with **no internet exposure anywhere**. Management access to each EC2 instance is done exclusively through **AWS Systems Manager Session Manager**, which reaches the instance over AWS PrivateLink (VPC Interface Endpoints), so no SSH key pair, no bastion host, no Internet Gateway, and no public IP is used at any point.

By the end of this lab you will have:

- Two AWS accounts, each with one fully private VPC, using **non-overlapping CIDR blocks**
- One private subnet in each VPC (no public subnet — it is not needed and is intentionally removed)
- One private EC2 instance in each VPC, with **no public IP, no SSH key pair, no bastion**
- Each EC2 instance managed exclusively via **AWS Systems Manager Session Manager**
- The three VPC Interface Endpoints required for SSM to function with zero internet access
- One Transit Gateway per account, peered cross-account
- VPC route tables and TGW route tables correctly configured so that **only the two private CIDR blocks can reach each other** — nothing else is reachable, and nothing outside can reach in
- Verified **bidirectional ping** between the two private EC2 instances, tested from inside a Session Manager session (never from your laptop, never over the internet)

---

## 2. Architecture Diagram (Overview)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/55309c16-88a1-42c1-b453-bfcd650f547d" />


---

## 3. Prerequisites

| # | Requirement | Why it matters |
|---|---|---|
| 1 | Two separate AWS accounts, with console access to both | You alternate between Account A and Account B throughout |
| 2 | IAM permissions in both accounts for VPC, EC2, IAM (to create a role), Transit Gateway, and Systems Manager | Needed to create every resource in this guide |
| 3 | The **12-digit AWS Account ID** of both accounts, noted down | Required for the cross-account peering attachment |
| 4 | Both accounts operating in the **same region** (`ap-south-1` used throughout) | Keeps the lab simple; TGW peering also supports cross-region |
| 5 | Two non-overlapping CIDR ranges chosen in advance | The most common reason TGW peering labs fail |
| 6 | No SSH key pair needed anywhere in this lab | Access is exclusively via Session Manager |

---

## 4. IP Addressing Plan

| Resource | Account | CIDR / Value | AZ |
|---|---|---|---|
| VPC-A | Account A | `10.10.0.0/16` | ap-south-1 |
| Private Subnet A | Account A | `10.10.2.0/24` | ap-south-1a |
| Private-EC2-A | Account A | e.g. `10.10.2.10` | ap-south-1a |
| SSM Interface Endpoints A | Account A | ENIs land inside `10.10.2.0/24` | ap-south-1a |
| TGW-A | Account A | Custom ASN `64512` | ap-south-1 |
| VPC-B | Account B | `10.20.0.0/16` | ap-south-1 |
| Private Subnet B | Account B | `10.20.2.0/24` | ap-south-1a |
| Private-EC2-B | Account B | e.g. `10.20.2.10` | ap-south-1a |
| SSM Interface Endpoints B | Account B | ENIs land inside `10.20.2.0/24` | ap-south-1a |
| TGW-B | Account B | Custom ASN `64513` (must differ from TGW-A's ASN, per AWS's recommendation for peered TGWs) | ap-south-1 |

**Verification check:** `10.10.0.0/16` and `10.20.0.0/16` share no address space — confirmed non-overlapping. ✔

---

## 5. Part A — Account A: VPC, Subnet, IAM Role, SSM Endpoints, EC2

Perform all of Part A while logged in to **Account A**.

### A1. Create VPC-A

1. VPC console → **Your VPCs** → **Create VPC**.
2. Select **VPC only**.
3. Name tag: `VPC-A`.
4. IPv4 CIDR block: `10.10.0.0/16`.
5. Tenancy: **Default**.
6. Choose **Create VPC**.

### A2. Create Private Subnet A

1. **Subnets** → **Create subnet**.
2. VPC: `VPC-A`.
3. Subnet name: `Private-Subnet-A`.
4. Availability Zone: `ap-south-1a`.
5. IPv4 CIDR block: `10.10.2.0/24`.
6. Choose **Create subnet**.
7. Confirm **Auto-assign public IPv4 address** is **disabled** (this is the default for a new subnet).

There is **no public subnet, no Internet Gateway, and no NAT Gateway anywhere in Account A**. This is intentional — the VPC is fully private end to end.

### A3. Create the Private Route Table

1. **Route Tables** → **Create route table**.
2. Name: `Private-RT-A`.
3. VPC: `VPC-A` → **Create route table**.
4. **Subnet associations** tab → **Edit subnet associations** → select `Private-Subnet-A` → **Save associations**.
5. Leave the route table with only its default **local** route for now — the TGW route is added later in Part F.

### A4. Create the IAM Role for SSM

1. **IAM console** → **Roles** → **Create role**.
2. Trusted entity type: **AWS service**.
3. Use case: **EC2** → **Next**.
4. Attach permissions policy: search for and select **`AmazonSSMManagedInstanceCore`** → **Next**.
5. Role name: `SSM-Instance-Role-A`.
6. **Create role**.

This managed policy grants exactly the permissions the SSM Agent on the instance needs to register with Systems Manager and accept Session Manager connections — no broader EC2 permissions are needed for this lab.

### A5. Create the Endpoint Security Group

1. **VPC console** → **Security Groups** → **Create security group**.
2. Name: `SSM-Endpoint-SG-A`, VPC: `VPC-A`.
3. Inbound rule: Type `HTTPS`, Port `443`, Source: `10.10.0.0/16` (VPC-A's own CIDR — this is what lets the private EC2 instance reach the endpoints).
4. Outbound: leave default (all traffic allowed).
5. **Create security group**.

### A6. Create the Private EC2 Security Group

1. **Create security group**.
2. Name: `Private-SG-A`, VPC: `VPC-A`.
3. Inbound rule: Type `All ICMP - IPv4`, Source: `10.20.0.0/16` (VPC-B's CIDR — this is the **only** inbound rule on this instance; there is deliberately no SSH rule at all).
4. Outbound: leave default (all traffic allowed) — this lets the instance's SSM Agent reach the VPC endpoints over HTTPS.
5. **Create security group**.

**This is the control point for "only the two VPCs can ping each other, not from anywhere else":** the private EC2's Security Group accepts ICMP from exactly one CIDR (the peer VPC) and nothing else, and has no SSH/RDP rule of any kind — it cannot be reached by any inbound connection except that one ICMP path.

### A7. Create the Three VPC Interface Endpoints for SSM

Create each of the following three endpoints identically, changing only the service name:

1. **VPC console** → **Endpoints** → **Create endpoint**.
2. Name: `vpce-ssm-A`.
3. Service category: **AWS services**.
4. Service Name: search for and select `com.amazonaws.ap-south-1.ssm`.
5. VPC: `VPC-A`.
6. Subnets: select `Private-Subnet-A` (choose the AZ `ap-south-1a` row).
7. Security group: select `SSM-Endpoint-SG-A` (deselect the default SG if it's pre-checked).
8. Policy: leave **Full access**.
9. **Additional settings** → confirm **Enable DNS name** is checked. This is essential — without it, the SSM Agent's default API calls will not resolve to the private endpoint IP.
10. Choose **Create endpoint**.

Repeat the exact same steps twice more for:

- `com.amazonaws.ap-south-1.ssmmessages` (name it `vpce-ssmmessages-A`)
- `com.amazonaws.ap-south-1.ec2messages` (name it `vpce-ec2messages-A`)

**Verification check:** After a few minutes, all three endpoints should show **State: Available** in the VPC console's Endpoints list, each with an ENI inside `10.10.2.0/24`.

### A8. Launch Private-EC2-A

1. **EC2 console** → **Launch instance**.
2. Name: `Private-EC2-A`.
3. AMI: **Amazon Linux 2023** (ships with the SSM Agent pre-installed).
4. Instance type: `t3.micro`.
5. Key pair: select **"Proceed without a key pair"** — no SSH key is used anywhere in this lab.
6. Network settings → Edit:
   - VPC: `VPC-A`
   - Subnet: `Private-Subnet-A`
   - Auto-assign public IP: **Disable**
   - Security group: `Private-SG-A`
7. **Advanced details** → **IAM instance profile**: select `SSM-Instance-Role-A`.
8. **Launch instance**.
9. Once running, note its **private IPv4 address** (e.g. `10.10.2.10`).

**Verification check:** In the **AWS Systems Manager console** → **Fleet Manager** (or **Session Manager** → **Start session**, target instances list), `Private-EC2-A` should appear with **Node status: Online** within a couple of minutes of launch. If it does not appear, stop here and revisit A4–A7 before continuing — everything downstream depends on this working.

---

## 6. Part B — Account B: VPC, Subnet, IAM Role, SSM Endpoints, EC2

Log out of Account A and log in to **Account B**. Repeat the same steps as Part A, substituting Account B's values from Section 4.

### B1. Create VPC-B
Same as A1: name `VPC-B`, CIDR `10.20.0.0/16`.

### B2. Create Private Subnet B
Same as A2: name `Private-Subnet-B`, AZ `ap-south-1a`, CIDR `10.20.2.0/24`. No public subnet, no IGW, no NAT — same as Account A.

### B3. Create Private-RT-B
Same as A3: associate with `Private-Subnet-B`; only the local route for now.

### B4. Create the IAM Role for SSM
Same as A4: role name `SSM-Instance-Role-B`, with `AmazonSSMManagedInstanceCore` attached.

### B5. Create the Endpoint Security Group
Same as A5: name `SSM-Endpoint-SG-B`, inbound HTTPS/443 from `10.20.0.0/16`.

### B6. Create the Private EC2 Security Group
Same as A6: name `Private-SG-B`, inbound **All ICMP - IPv4** from `10.10.0.0/16` (VPC-A's CIDR) only. No SSH rule.

### B7. Create the Three VPC Interface Endpoints
Same as A7, in `VPC-B` / `Private-Subnet-B` / `SSM-Endpoint-SG-B`, with **Enable DNS name** checked:
- `com.amazonaws.ap-south-1.ssm` → `vpce-ssm-B`
- `com.amazonaws.ap-south-1.ssmmessages` → `vpce-ssmmessages-B`
- `com.amazonaws.ap-south-1.ec2messages` → `vpce-ec2messages-B`

### B8. Launch Private-EC2-B
Same as A8: no key pair, no public IP, subnet `Private-Subnet-B`, security group `Private-SG-B`, IAM instance profile `SSM-Instance-Role-B`. Note its private IP (e.g. `10.20.2.10`).

**Verification check:** `Private-EC2-B` should show **Node status: Online** in Account B's Systems Manager console.

**Checkpoint:** You now have two fully independent, fully private VPCs, each with an SSM-managed instance reachable only through Session Manager, but **no connectivity between them yet** — that's expected, the TGW peering has not been built.

---

## 7. Part C — Create Transit Gateways (One per Account)

### C1. Create TGW-A (in Account A)

1. Switch to **Account A**, VPC console → **Transit Gateways** → **Create transit gateway**.
2. Name: `TGW-A`.
3. Description: `TGW for Account A`.
4. Amazon side ASN: set to `64512`. **AWS recommends giving each peered transit gateway a unique ASN**, so note this value — TGW-B must be assigned a different one.
5. DNS support: leave **Enabled**.
6. VPN ECMP support: leave **Enabled** (unused in this lab, harmless to leave on).
7. Default route table association: **Enable**.
8. Default route table propagation: **Enable**.
9. Choose **Create transit gateway**. Wait for state `available`.
10. Note the **Transit Gateway ID** (`tgw-xxxxxxxx`).

### C2. Create TGW-B (in Account B)

1. Switch to **Account B**, same console path.
2. Name: `TGW-B`, description `TGW for Account B`.
3. Amazon side ASN: set to `64513` — must differ from TGW-A's `64512`.
4. Same other defaults as C1 (DNS support enabled, default route table association + propagation enabled).
5. **Create transit gateway**, wait for `available`.
6. Note the **Transit Gateway ID** for TGW-B.

**Verification check:** Two independent TGWs, each with its own auto-created default route table, neither with any attachments yet.

---

## 8. Part D — Attach Each VPC to Its Own TGW

### D1. Attach VPC-A to TGW-A (in Account A)

1. Account A, VPC console → **Transit Gateway Attachments** → **Create transit gateway attachment**.
2. Transit gateway ID: `TGW-A`.
3. Attachment type: **VPC**.
4. Name tag: `VPC-A-Attachment`.
5. DNS support: **Enable**.
6. VPC ID: `VPC-A`.
7. Subnet IDs: select `Private-Subnet-A`.
8. **Create transit gateway attachment**, wait for `available`.

### D2. Attach VPC-B to TGW-B (in Account B)

1. Account B, same path.
2. Transit gateway ID: `TGW-B`, name tag `VPC-B-Attachment`.
3. VPC ID: `VPC-B`, Subnet IDs: `Private-Subnet-B`.
4. **Create transit gateway attachment**, wait for `available`.

**Verification check:** Both attachments show `available`, and both appear associated + propagated in their own TGW's default route table (Section C left those settings enabled).

---

## 9. Part E — Cross-Account TGW Peering Attachment

### E1. Create the Peering Attachment (Requester — Account A)

1. Account A, VPC console → **Transit Gateway Attachments** → **Create transit gateway attachment**.
2. Transit gateway ID: `TGW-A`.
3. Attachment type: **Peering Connection**.
4. Name tag: `TGW-A-to-TGW-B-Peering`.
5. Account: **Other account** → Account ID: **Account B's 12-digit account ID**.
6. Region: `ap-south-1`.
7. Transit gateway (accepter): **TGW-B's Transit Gateway ID**.
8. **Create transit gateway attachment**. State shows `pendingAcceptance`.

### E2. Accept the Peering Attachment (Accepter — Account B)

1. Account B, VPC console → **Transit Gateway Attachments**.
2. Select the incoming attachment (`pendingAcceptance`) → **Actions** → **Accept transit gateway attachment**.
3. Confirm — state becomes `available` in both accounts.

> **Note on AWS RAM:** not required here — each account keeps its own TGW; only the two TGWs are peered as independent resources.

**Verification check:** The peering attachment shows `available` in both accounts' Transit Gateway Attachments lists.

---

## 10. Part F — VPC Route Table Updates (Post-Peering)

### F1. Update Private-RT-A (Account A)

1. Account A, `Private-RT-A` → **Edit routes** → **Add route**:
   - Destination: `10.20.0.0/16`
   - Target: **Transit Gateway** → `TGW-A`
2. **Save changes**.

### F2. Update Private-RT-B (Account B)

1. Account B, `Private-RT-B` → **Edit routes** → **Add route**:
   - Destination: `10.10.0.0/16`
   - Target: **Transit Gateway** → `TGW-B`
2. **Save changes**.

**Verification check:** `Private-RT-A` has `10.20.0.0/16 → TGW-A`; `Private-RT-B` has `10.10.0.0/16 → TGW-B`. Neither route table has, or needs, a `0.0.0.0/0` route — there is no internet path in this lab at all.

---

## 11. Part G — TGW Route Table Updates (Static Routes)

Peering attachments **do not support automatic route propagation** — a static route must be added manually on each side.

### G1. Add a Static Route in TGW-A's Route Table (Account A)

1. Account A, VPC console → **Transit Gateway Route Tables** → select the default route table associated with `TGW-A`.
2. **Routes** tab → **Create static route**.
3. CIDR: `10.20.0.0/16`.
4. Attachment: `TGW-A-to-TGW-B-Peering`.
5. **Create static route**.

### G2. Add a Static Route in TGW-B's Route Table (Account B)

1. Account B, same path — TGW-B's default route table.
2. **Create static route**.
3. CIDR: `10.10.0.0/16`.
4. Attachment: the peering attachment (viewed from the B side).
5. **Create static route**.

**Verification check:** Each TGW route table shows one **propagated** route (its own local VPC attachment) and one **static** route (the remote CIDR → the peering attachment), both `active`.

---

## 12. Security Groups / NACL Considerations

| Component | Setting | Reasoning |
|---|---|---|
| `Private-SG-A` | Inbound: **only** `All ICMP - IPv4` from `10.20.0.0/16`. No SSH, no RDP, no other inbound rule. | This is the entire enforcement point for "only the two VPCs can ping each other, not from anywhere" — nothing else can initiate a connection to this instance |
| `Private-SG-B` | Inbound: **only** `All ICMP - IPv4` from `10.10.0.0/16` | Mirrors Private-SG-A for the return path |
| `SSM-Endpoint-SG-A` / `-B` | Inbound HTTPS/443 from the instance's own VPC CIDR only | Scopes the endpoints so only instances inside that specific VPC can use them — not a blanket `0.0.0.0/0` |
| Security Groups are stateful | Replies to allowed traffic are automatically permitted | No separate outbound ICMP-reply rule is needed |
| NACLs | Default NACL allows all inbound/outbound | If replaced with a custom NACL, explicitly allow inbound/outbound ICMP for the peer CIDR, and inbound/outbound TCP 443 (both directions, since NACLs are stateless) for the SSM Agent's traffic to the endpoints |
| Session Manager traffic itself | Never touches the EC2 Security Group's inbound rules | The Session Manager connection is always **initiated outbound** from the instance's SSM Agent to the VPC endpoint — this is why no inbound SSH/RDP rule is ever required for SSM access |

---

## 13. Complete Packet Flow Walkthrough

### 13.1 Management path (Session Manager) — for reference

1. You open **Session Manager** in the AWS Console and choose `Private-EC2-A`.
2. AWS Systems Manager sends the session request to the **SSM Agent** running on the instance — this arrives via the `ssmmessages` VPC endpoint, entirely inside VPC-A; no public IP or inbound port on the instance is involved.
3. The SSM Agent, which is continuously polling **outbound** to the `ssm` and `ec2messages` endpoints, picks up the session and opens a local shell.
4. Your keystrokes and the shell's output are relayed back over that same private, encrypted channel.

### 13.2 Data path (ICMP ping) — the actual lab test

Trace of a single ICMP echo request sent from **Private-EC2-A** (`10.10.2.10`, inside a Session Manager shell) to **Private-EC2-B** (`10.20.2.10`):

1. Inside the Session Manager shell on Private-EC2-A, you run `ping 10.20.2.10`.
2. The packet leaves EC2-A's ENI, is permitted by `Private-SG-A`'s outbound rule (allow-all) and the subnet's default NACL.
3. **Private-RT-A** is consulted: most specific match for `10.20.2.10` is `10.20.0.0/16 → TGW-A` (Part F). Packet goes to the Transit Gateway.
4. **TGW-A**'s route table: static route `10.20.0.0/16 → peering attachment` (Part G) matches. Packet crosses the peering link.
5. The packet crosses the **cross-account TGW peering link**, staying entirely on the AWS backbone.
6. **TGW-B**'s route table: propagated route `10.20.0.0/16 → VPC-B attachment` matches. Packet is forwarded into VPC-B.
7. **Private-RT-B** delivers the packet to Private-EC2-B's ENI.
8. The subnet NACL (default allow) and `Private-SG-B`'s inbound rule (`All ICMP from 10.10.0.0/16`) both permit it.
9. Private-EC2-B replies with an ICMP echo reply, retracing the same path in reverse; the reply is automatically permitted back through `Private-SG-A` because Security Groups are stateful.

If any single step fails — a missing route, a missing static TGW route, a wrong SG rule, or the SSM Agent itself being offline — the fault isolates to that exact hop. This is the basis for Section 15.

---

## 14. Connecting via Session Manager & Ping Testing — Both Directions

No SSH key, no bastion, and no public IP are used anywhere in this section — everything happens inside the AWS Console.

### 14.1 Connect to Private-EC2-A

1. **EC2 console** (Account A) → **Instances** → select `Private-EC2-A`.
2. Choose **Connect** → **Session Manager** tab → **Connect**.
3. A browser-based shell opens directly on the instance.

### 14.2 Connect to Private-EC2-B

1. **EC2 console** (Account B) → **Instances** → select `Private-EC2-B`.
2. **Connect** → **Session Manager** tab → **Connect**.

### 14.3 Test: Account A → Account B

From the Session Manager shell on **Private-EC2-A**, run:
```
ping 10.20.2.10
```
Expected result: replies with `64 bytes from 10.20.2.10: icmp_seq=... ttl=... time=...`, 0% packet loss.

### 14.4 Test: Account B → Account A

From the Session Manager shell on **Private-EC2-B**, run:
```
ping 10.10.2.10
```
Expected result: same as above, in reverse — confirming the path is symmetric.

### 14.5 Confirm nothing else can reach either instance

From your own laptop (outside both VPCs entirely), attempting to `ping` or `ssh` to either instance's private IP will simply fail — there is no route from the public internet to `10.10.2.0/24` or `10.20.2.0/24` at all, and even if there were, `Private-SG-A`/`Private-SG-B` accept ICMP from only the peer VPC's CIDR and have no SSH rule whatsoever. This is what confirms the "only the two VPCs can ping each other, not from anywhere" requirement.

**Successful outcome:** Both directions inside Session Manager show 0% packet loss; nothing outside the two VPCs can reach either instance by any protocol.

---

## 15. Troubleshooting Checklist

| # | Check | How to verify | Fix |
|---|---|---|---|
| 1 | Instance not appearing in Systems Manager / "Node status: Offline" | Systems Manager console → Fleet Manager | Confirm the IAM instance profile is attached (Section A4/A8), and that all three VPC endpoints exist and are `Available` (Section A7) |
| 2 | Session Manager "Connect" button greyed out or fails | EC2 console → instance → Connect → Session Manager tab | Same causes as #1 — the SSM Agent must be able to reach `ssm`, `ssmmessages`, and `ec2messages` |
| 3 | Endpoint Security Group too narrow | `SSM-Endpoint-SG-A`/`-B` inbound rules | Must allow HTTPS/443 from the instance's own VPC CIDR (not from the *peer* VPC's CIDR — that's a different rule on the EC2 SG) |
| 4 | "Enable DNS name" not checked on an endpoint | VPC console → Endpoints → select endpoint → Details | Without private DNS, the instance's calls to the public SSM API hostnames won't resolve to the private endpoint ENIs; edit the endpoint and enable it |
| 5 | CIDR overlap | Compare VPC-A and VPC-B CIDRs | Must be redesigned if they overlap — cannot be fixed with routing alone |
| 6 | Peering attachment state | Transit Gateway Attachments, both accounts | Must read `available` in both; `pendingAcceptance` means Account B hasn't accepted yet (E2) |
| 7 | Wrong accepter TGW ID or account ID | Re-check values entered in E1 | Delete and recreate the peering attachment with the correct IDs |
| 8 | VPC route table missing remote CIDR | Private-RT-A / Private-RT-B → Routes tab | Add the route to the peer VPC's CIDR with target = local TGW (Part F) |
| 9 | TGW route table missing static route | Transit Gateway Route Tables → each TGW's RT → Routes tab | Add the static route to the peering attachment (Part G) — this never auto-propagates |
| 10 | TGW static route pointing at the wrong attachment | Same screen as #9 | Must point at the *peering* attachment, not the local VPC attachment |
| 11 | ICMP blocked by Security Group | `Private-SG-A`/`-B` inbound rules | Confirm the ICMP rule's source is the **peer** VPC's CIDR, not its own |
| 12 | NACL blocking traffic (only if the default NACL was replaced) | Subnet's associated NACL | Re-check inbound/outbound ICMP for the peer CIDR |
| 13 | Region mismatch | Confirm both TGWs and the peering attachment are in the same expected region | A same-region lab referencing a different accepter-TGW region will fail attachment creation |
| 14 | TGW route table association | Transit Gateway Route Tables → Associations tab | The VPC attachment must be associated with the route table you're editing |

---

## 16. VPC Peering vs. Transit Gateway Peering — Comparison

| Aspect | VPC Peering | Transit Gateway Peering |
|---|---|---|
| What is peered | Two VPCs directly | Two Transit Gateways (each can front many VPCs) |
| Topology at scale | Full mesh required — N VPCs need N(N-1)/2 peering connections | Hub-and-spoke on each side; one peering link regardless of VPC count behind each TGW |
| Transitive routing | Not supported | Also not supported across the peering link itself — only directly propagated/static routes cross |
| Cross-account support | Yes | Yes |
| Cross-region support | Yes | Yes |
| Route propagation | Manual entries in each VPC route table | VPC attachments can auto-propagate into the TGW route table; peering attachments require a manual static route |
| Bandwidth | No published aggregate limit; scales with the instances involved | Per-attachment bandwidth burst limits published by AWS |
| Pricing model | No hourly charge for the peering connection; pay for data transfer | Hourly charge per TGW attachment plus per-GB data processing |
| Management overhead | Low for 2 VPCs, grows quickly beyond a handful | Low regardless of scale — new VPCs just attach to their local TGW |
| Best fit | Small number of VPCs, simple two-account connectivity, cost-sensitive | Many VPCs per side, hub-and-spoke design, connecting two large environments |

---

