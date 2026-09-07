# Point-to-Point VPN vs Site-to-Site VPN — Complete Guide

A plain-language, no-gaps reference covering the concept of VPN connectivity, how Point-to-Point differs from Site-to-Site, and how AWS implements Site-to-Site VPN as a managed service — including every component, billing variable, and the trade-offs of each approach.

---

## 1. The scenario

You run a company with:
- An **on-premises data center** (or a branch office) with private servers — ERP, file shares, internal databases.
- Some workloads now live in an **AWS VPC**.
- The two sides need to talk to each other **privately**, over the **public internet**, without exposing anything publicly, and without waiting weeks for a leased line.

This is the exact problem VPNs solve. The question is *which kind* of VPN fits your topology — one office reaching one destination (Point-to-Point), or many private networks reaching each other in a mesh or hub (Site-to-Site).

---

## 2. What is a VPN (the base concept)

A **VPN (Virtual Private Network)** creates an **encrypted, authenticated tunnel** across a network you don't trust (usually the public internet), so two private networks — or a device and a private network — can exchange traffic as if a private, dedicated cable connected them.

Three things a VPN guarantees:

| Property | What it means |
|---|---|
| **Confidentiality** | Traffic is encrypted (commonly via IPsec) — unreadable if intercepted in transit. |
| **Integrity** | Packets are authenticated; tampering in transit is detected and the packet is dropped. |
| **Tunneling** | Private IP ranges on both ends reach each other without being exposed to the public internet. |

VPNs are built on protocols like **IPsec** (IKE for key exchange, ESP for encrypting the payload), or newer options like **WireGuard**. AWS Site-to-Site VPN uses IPsec.

---

## 3. Point-to-Point VPN — what it is

**Point-to-Point VPN (P2P VPN)** connects **exactly two endpoints** — typically one remote device or one single network to one destination network. Think: a remote worker's laptop connecting to a corporate network, or one branch office connecting to one head office.

**Key traits:**
- One tunnel, one relationship: **Point A ↔ Point B**.
- Usually **client-to-site** (a single device dialing in) rather than network-to-network, though it can also be network-to-network if there are only two sites total.
- Simple routing — there's only one other end to route to.
- AWS's equivalent of this pattern is **AWS Client VPN** (an individual user's device connects into a VPC) — not Site-to-Site VPN.

**Typical use case:** a remote employee needs secure access to internal resources from home, or a single small office needs to reach one central VPC.

---

## 4. Site-to-Site VPN — what it is

**Site-to-Site VPN (S2S VPN)** connects **entire networks to entire networks** — not a single device, but every host on one private network to every host on another, transparently, through gateways that sit at the edge of each network.

**Key traits:**
- Connects **two networks** (or many, in a hub-and-spoke), not two individual devices.
- Traffic from *any* host on Network A can reach *any* host on Network B (subject to routing and security group / firewall rules) — no per-device VPN client needed.
- Terminated by **gateway devices** on each side (a router/firewall on-prem, a managed gateway in AWS), not by end-user software.
- Scales to many sites — this is what lets a company connect 10 branch offices to one AWS VPC, or connect several VPCs and several on-prem sites together via a hub.

**Point-to-Point vs Site-to-Site, side by side:**

| | Point-to-Point VPN | Site-to-Site VPN |
|---|---|---|
| Connects | One device ↔ one network, or one site ↔ one site | Entire network ↔ entire network |
| Who initiates | End-user client software | Gateway devices (routers/firewalls) |
| Scale | Single relationship | Can scale to many sites (hub-and-spoke) |
| AWS equivalent | AWS Client VPN | AWS Site-to-Site VPN |
| Typical user | Remote employee, single branch | Whole offices, data centers, multi-site enterprises |
| Routing complexity | Minimal — one path | Needs route tables / BGP across all sites |

---

## 5. Why use a VPN over a public, unencrypted connection

- **No new physical circuit** — runs over your existing internet connection.
- **Encryption in transit** — meets most compliance baselines for data crossing a public network.
- **Private IP addressing end-to-end** — internal hosts never need public IPs or exposed ports.
- **Fast to provision** — hours, not the weeks a dedicated line (like AWS Direct Connect) can take.
- **Lower cost** than dedicated circuits, for moderate and non-latency-critical throughput.

---

## 6. AWS services involved

| Service | What it's for |
|---|---|
| **AWS Site-to-Site VPN** | The core managed IPsec VPN service — connects an on-prem network (or another cloud) to a VPC or Transit Gateway. |
| **AWS Client VPN** | The Point-to-Point equivalent — lets individual users' devices connect securely into a VPC using an OpenVPN-based client. |
| **Virtual Private Gateway (VGW)** | The original AWS-side VPN endpoint, attached to a single VPC. |
| **Transit Gateway (TGW)** | A newer, more scalable AWS-side endpoint — terminates VPN connections and fans them out to many attached VPCs, other VPNs, and Direct Connect at once. |
| **AWS Direct Connect** | Not a VPN — a dedicated private physical circuit. Often combined *with* Site-to-Site VPN (as a backup path, or to encrypt traffic over DX itself via VPN-over-DX). |
| **CloudWatch** | Monitors tunnel status (UP/DOWN), data in/out, and tunnel state changes — used for alerting on tunnel failover. |

---

## 7. Components of an AWS Site-to-Site VPN connection

| Component | Role |
|---|---|
| **Customer Gateway (CGW)** | An AWS resource representing your on-prem (or other-cloud) router — its public static IP, and its BGP ASN if using dynamic routing. Doesn't cost anything by itself — it's just a config object. |
| **Virtual Private Gateway (VGW)** | The AWS-side endpoint attached to one VPC. The "classic" way to terminate a VPN. |
| **Transit Gateway (TGW)** | Alternative AWS-side endpoint. Preferred in multi-VPC / multi-site environments because one TGW can terminate many VPN connections and route between all attached VPCs. |
| **VPN Connection** | The logical AWS resource that joins a CGW to a VGW or TGW. Creating one automatically provisions **two tunnels**. |
| **Tunnels (×2 per connection)** | Two independent IPsec tunnels, each terminating in a different AWS Availability Zone, for redundancy — if one AWS endpoint has an issue, the second tunnel keeps traffic flowing. |
| **Pre-shared key (PSK) or certificate** | The authentication method (IKE Phase 1) used to establish each tunnel — a PSK is simplest, certificate-based auth is also supported. |
| **Routing (Static or BGP)** | Static: you manually list the CIDRs reachable on each side. BGP (dynamic): routes are exchanged automatically over the tunnel — preferred because it enables automatic failover between the two tunnels without manual intervention. |
| **VPC Route Tables** | Must have a route pointing your on-prem CIDR at the VGW or TGW — otherwise return traffic from the VPC never finds its way back to the tunnel. |
| **On-prem router / firewall** | The physical or virtual device (Cisco ASA, Palo Alto, StrongSwan, Openswan, pfSense, etc.) that actually terminates the tunnel on your side — must match the IKE/IPsec parameters AWS expects. |
| **Security groups / NACLs** | Don't forget — a working tunnel doesn't bypass VPC-level firewalling. Instances still need inbound rules allowing traffic from the on-prem CIDR. |

---

## 8. How it fits together (architecture)

```
On-premises                          Public Internet                    AWS
┌─────────────────────┐                                        ┌──────────────────────────┐
│  Private network      │                                       │  VPC (10.20.0.0/16)      │
│  10.0.0.0/16           │                                       │                          │
│                        │                                       │  ┌────────────────────┐  │
│  ┌──────────────────┐  │      Tunnel 1 (AZ-a) ───────────────▶│  │ Virtual Private     │  │
│  │ Customer Gateway  │──┼──────────────────────────────────────▶ │ Gateway (VGW)       │  │
│  │ (on-prem router)  │  │      Tunnel 2 (AZ-b) ───────────────▶│  │  or Transit Gateway │  │
│  └──────────────────┘  │                                       │  └──────────┬─────────┘  │
│  static public IP      │                                       │             │            │
└─────────────────────┘                                        │   route table → VGW/TGW    │
                                                                  │             ▼            │
                                                                  │  ┌────────────────────┐  │
                                                                  │  │ Private subnet     │  │
                                                                  │  │ EC2 / RDS / etc.   │  │
                                                                  │  └────────────────────┘  │
                                                                  └──────────────────────────┘
```

Both tunnels are active endpoints from AWS's side — either can carry traffic. With BGP, AWS advertises both paths and the on-prem router picks the best/available one automatically. With static routing, you configure both but typically only one carries traffic unless you script failover.

---

## 9. Billing variables — what actually costs money

This is the part people miss until the bill arrives. Breaking it down:

| Cost component | How it's billed | Notes |
|---|---|---|
| **VPN Connection — hourly charge** | Per VPN connection, per hour it exists (whether or not it's passing traffic) | Charged from the moment the connection is created until deleted, regardless of tunnel UP/DOWN state. |
| **Data processed through the VPN connection** | Per GB, for data going *through* the VPN connection (in and out) | This is on top of the hourly charge — check current AWS pricing page for the per-GB rate in your region. |
| **Transit Gateway — attachment hourly charge** | Per TGW attachment, per hour (if using TGW instead of VGW) | A VPN attachment to a TGW is billed separately from the VPN connection's own hourly charge — this stacks. |
| **Transit Gateway — data processing charge** | Per GB processed through the TGW | Separate from the VPN's own per-GB charge if you route through TGW — the same byte can be billed once for VPN processing and again for TGW processing. |
| **Data transfer OUT to the internet** | Standard AWS data-transfer-out rates | Only applies where relevant — the VPN tunnel traffic itself isn't "internet egress" in the traditional sense, but check your architecture for any double-hop scenarios. |
| **VGW** | No separate hourly charge for the VGW resource itself (unlike TGW) | The VPN connection's hourly rate is the main cost when using VGW. |
| **Client VPN (Point-to-Point) — endpoint hourly charge** | Per VPN endpoint association, per hour | Separate pricing model from Site-to-Site — billed per subnet association plus a per-connection hourly rate. |
| **CloudWatch charges** | Standard CloudWatch metrics/alarms pricing | Small, but adds up if you're polling tunnel state frequently or storing custom metrics. |

**Practical implication:** a VPN connection you forgot to delete after a test still bills hourly, even with zero traffic. Always check `describe-vpn-connections` before assuming cost is zero.

---

## 10. Advantages

- **Fast to provision** — a working tunnel in hours, no physical circuit needed.
- **Encrypted by default** — IPsec protects data in transit without extra effort.
- **Redundant by design** — two tunnels across two AZs, out of the box, for every connection.
- **Cost-effective** for low-to-moderate, non-latency-sensitive throughput compared to Direct Connect.
- **No new hardware in AWS** — the AWS side is fully managed (VGW/TGW); you only manage your on-prem router config.
- **Works over any existing internet connection** — no ISP coordination required beyond what you already have.
- **BGP support** gives automatic failover between tunnels without manual scripting.

## 11. Disadvantages

- **Throughput ceiling** — each individual tunnel is capped (historically around 1.25 Gbps per tunnel); high-throughput workloads need Direct Connect, ECMP across multiple VPN connections, or ECMP with TGW.
- **Variable latency and jitter** — it's still riding the public internet, so performance isn't guaranteed like a dedicated circuit.
- **Depends on internet reachability** — an ISP outage on either side takes the tunnel down; there's no SLA on the underlying internet path itself.
- **On-prem router compatibility matters** — older or non-standard routers may not support the exact IKE/IPsec parameters AWS requires, causing tunnel negotiation failures.
- **Static routing doesn't fail over automatically** — without BGP, a tunnel outage requires manual intervention or scripted route changes.
- **Costs stack in TGW architectures** — VPN hourly + TGW attachment hourly + two layers of per-GB processing can surprise teams used to VGW's simpler billing.
- **Not ideal for latency-sensitive or very high-bandwidth production workloads** — those belong on Direct Connect.

---

## 12. Common edge cases and gotchas

| Situation | What actually happens |
|---|---|
| CIDR overlap between on-prem and VPC | Routing breaks — VPN doesn't NAT by default, so both sides must use non-overlapping CIDR ranges. |
| Only one tunnel configured on-prem | You lose the redundancy AWS designed in — always terminate both tunnels if your router supports it. |
| Static routing, no failover script | If the active tunnel drops, traffic simply stops until someone notices and manually intervenes. |
| VPC route table missing the on-prem route | Outbound reaches on-prem fine, but return traffic has nowhere to go — looks like a one-way connectivity issue. |
| Security groups too restrictive | Tunnel shows UP in both consoles, but application traffic still fails — always double-check SGs/NACLs separately from tunnel status. |
| Forgetting to delete test VPN connections | Hourly billing continues even with zero data flowing. |
| Mixing VGW and TGW in the same account without a clear reason | Adds architectural complexity and possible duplicate billing paths — pick one model per environment. |

---

## 13. Interview-style Q&A

**Q: What's the fundamental difference between Point-to-Point and Site-to-Site VPN?**
A: Point-to-Point connects a single device or single site to one destination; Site-to-Site connects entire networks to entire networks via gateways, so any host on either side can reach any host on the other without per-device client software.

**Q: Why does AWS create two tunnels per VPN connection?**
A: For redundancy — the two tunnels terminate in different Availability Zones on the AWS side, so an AZ-level issue on AWS's end doesn't take down connectivity if the on-prem router is configured to use both.

**Q: What's the difference between VGW and TGW as a VPN endpoint?**
A: VGW attaches to exactly one VPC and has no separate hourly charge. TGW is a standalone routing hub that can terminate many VPN connections, VPC attachments, and Direct Connect connections at once, but it adds its own hourly attachment and data-processing charges.

**Q: Static vs BGP routing — when would you choose each?**
A: Static routing is simpler for a small, fixed set of CIDRs and doesn't require BGP-capable hardware, but requires manual failover. BGP is preferred whenever the on-prem router supports it, since it gives automatic failover between tunnels and can propagate route changes without reconfiguration.

**Q: What causes a VPN connection to bill even when idle?**
A: The hourly charge is for the *existence* of the VPN connection resource, not for traffic — it accrues regardless of tunnel state or data volume.

**Q: How would you troubleshoot a tunnel that's UP but an application still can't connect?**
A: Check in order: VPC route table has the on-prem CIDR pointed at the VGW/TGW; security groups and NACLs allow the traffic; the on-prem firewall isn't blocking return traffic; and confirm there's no CIDR overlap causing silent routing conflicts.

---

## 14. Cheat sheet

- **P2P VPN** = one device/site ↔ one destination → AWS Client VPN.
- **S2S VPN** = network ↔ network via gateways → AWS Site-to-Site VPN.
- Every AWS VPN connection = **2 tunnels**, 2 AZs, for redundancy.
- **VGW** = simple, one VPC, no separate hourly fee.
- **TGW** = scalable, many VPCs/VPNs, has its own hourly + data fee.
- **BGP > static** for automatic failover.
- CIDRs on both sides must **not overlap**.
- Billing = **VPN hourly** + **per-GB processed** + (**TGW hourly** + **TGW per-GB**, if used).
- A tunnel showing UP doesn't guarantee the application works — check route tables and security groups separately.

---

## 15. Mastery checklist

- [ ] Can explain P2P vs S2S VPN to a non-technical stakeholder in two sentences.
- [ ] Can draw the AWS Site-to-Site VPN architecture from memory (CGW, tunnels, VGW/TGW, route table, subnet).
- [ ] Knows every billing line item and can estimate monthly cost for a given topology.
- [ ] Can list the difference between VGW-based and TGW-based designs, and when to pick each.
- [ ] Knows why two tunnels exist and what BGP adds over static routing.
- [ ] Can troubleshoot "tunnel UP, app still failing" without guessing.
