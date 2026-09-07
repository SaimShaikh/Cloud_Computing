# Point-to-Point, Point-to-Site, and Site-to-Site VPN — Complete Guide

A plain-language, no-gaps reference covering the concept of VPN connectivity, the three ways networks and devices connect over one, and how AWS implements Site-to-Site VPN as a managed service — including every component, billing variable, and the trade-offs of each approach.

---

## 1. The scenario

You run a company with:
- An **on-premises data center** (or a branch office) with private servers — ERP, file shares, internal databases.
- Some workloads now live in an **AWS VPC**.
- The two sides need to talk to each other **privately**, over the **public internet**, without exposing anything publicly, and without waiting weeks for a leased line.

This is the exact problem VPNs solve. The question is *which kind* of VPN fits your topology — a fixed link between exactly two endpoints (Point-to-Point), a single remote device reaching into a network (Point-to-Site), or many private networks reaching each other in a mesh or hub (Site-to-Site).

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

## 4. Point-to-Site VPN — what it is

**Point-to-Site VPN (P2S VPN)** connects **a single device (client) to an entire private network (site)** — the client gets a private IP address from the VPN server, and from that point on it can reach resources inside the target network as if it were physically plugged in.

This is the most common VPN pattern for **remote workers**: an employee on a laptop at home runs a VPN client, connects to the company's VPN endpoint, and their machine can now reach internal servers, databases, and services that are not exposed to the public internet at all.

**Key traits:**
- One side is always a **single device** (the client — laptop, phone, VM).
- The other side is a **network with a VPN endpoint/server** (the "site").
- The client needs to install and run **VPN client software** — nothing happens automatically.
- Each device that wants access must connect individually — there is no shared gateway that covers an entire office.
- The connection is usually **on-demand**: the user opens the VPN client, authenticates, the tunnel comes up, and it drops when the client disconnects.
- Scales by adding more client licences/connections on the server side — not by adding gateway devices.

**How it works — step by step:**

```
Remote Laptop (Client)                Public Internet            VPN Endpoint (Server / Site)
──────────────────────                ───────────────            ────────────────────────────

1. User opens VPN client app
2. Client sends IKE/TLS handshake ──────────────────────────▶  VPN server authenticates
3. Server assigns client a VIP:  ◀──────────────────────────   e.g. 172.16.0.50 (virtual IP)
4. Tunnel established

Now:
Laptop (192.168.1.20) ──[encrypted tunnel]──▶ VPN Server ──▶ Internal DB (10.0.2.30)

From the DB's perspective, the request came from 172.16.0.50 — it looks
like an internal address, not a remote laptop on the public internet.
```

**Authentication options:**

| Method | How it works | Strength |
|---|---|---|
| Username + Password | User provides credentials at connect time | Basic — weakest alone |
| Certificate (mutual TLS) | Client presents a certificate issued by your CA | Strong — no passwords |
| MFA (SAML / OIDC) | Integrates with an Identity Provider (Okta, Azure AD, etc.) | Strongest — phishing-resistant |
| Combined (cert + MFA) | Certificate AND identity provider login | Enterprise standard |

**Split tunnelling vs full tunnelling:**

```
Full tunnel (all traffic goes through VPN):
  Laptop ──[ALL traffic via VPN]──▶ VPN Server ──▶ internet + internal resources
  Benefit: Company controls all traffic, sees everything.
  Downside: Slower for the user (all YouTube traffic goes through corporate too).

Split tunnel (only internal traffic goes through VPN):
  Internal resources (10.0.0.0/16) ──[via VPN]──▶ VPN Server ──▶ Internal resources
  Internet traffic (0.0.0.0/0)     ──[direct]──▶  ISP ──▶ Internet
  Benefit: Faster for user, less load on VPN server.
  Downside: Company doesn't see or control internet browsing.
```

**AWS implementation — AWS Client VPN:**

AWS Client VPN is AWS's managed Point-to-Site service. Key facts:

| Property | Detail |
|---|---|
| **Protocol** | OpenVPN (TLS-based) — not IPsec |
| **Client software** | AWS-provided OpenVPN-compatible client, or any standard OpenVPN client |
| **Authentication** | Active Directory, SAML (Okta, Azure AD), certificate-based, or combined |
| **Endpoint attachment** | Client VPN endpoint attaches to a VPC — clients get routed into subnets |
| **Split tunnelling** | Supported — you define which routes go through the tunnel |
| **Authorization rules** | CIDR-level access rules per user group — Group A can reach 10.0.1.0/24, Group B cannot |
| **Logging** | Connection logs to CloudWatch — who connected, when, from what IP |
| **Billing** | Per endpoint-subnet association per hour + per active client connection per hour |

**AWS Client VPN architecture:**

```
Remote Laptop                Public Internet                      AWS
──────────────                ───────────────               ──────────────────────────────────
                                                            │   Client VPN Endpoint            │
┌──────────────┐                                           │   (attached to VPC subnet)        │
│ OpenVPN      │──────────── TLS tunnel ──────────────────▶│                                  │
│ client app   │                                           │   Authorization rules:            │
│ (AWS-provided│                                           │   - Devs → 10.0.1.0/24 (allowed) │
│  or OpenVPN) │                                           │   - Finance → 10.0.2.0/24 only   │
└──────────────┘                                           └──────────────┬───────────────────┘
Assigned VIP: 172.16.0.5                                                  │
                                                                          ▼
                                                           ┌──────────────────────────────────┐
                                                           │  VPC (10.0.0.0/16)               │
                                                           │  ┌──────────┐  ┌──────────────┐  │
                                                           │  │ Dev EC2  │  │ Finance RDS  │  │
                                                           │  │10.0.1.10 │  │ 10.0.2.20   │  │
                                                           │  └──────────┘  └──────────────┘  │
                                                           └──────────────────────────────────┘
```

**Typical use cases:**
- Developers working remotely who need SSH access to private EC2 instances
- Finance team accessing RDS databases that have no public endpoint
- IT admins managing internal AWS resources without bastion hosts
- Third-party contractors who need scoped, time-limited access to specific subnets

---

## 5. Site-to-Site VPN — what it is

**Site-to-Site VPN (S2S VPN)** connects **entire networks to entire networks** — not a single device, but every host on one private network to every host on another, transparently, through gateways that sit at the edge of each network.

**Key traits:**
- Connects **two networks** (or many, in a hub-and-spoke), not two individual devices.
- Traffic from *any* host on Network A can reach *any* host on Network B (subject to routing and security group / firewall rules) — no per-device VPN client needed.
- Terminated by **gateway devices** on each side (a router/firewall on-prem, a managed gateway in AWS), not by end-user software.
- Scales to many sites — this is what lets a company connect 10 branch offices to one AWS VPC, or connect several VPCs and several on-prem sites together via a hub.

**All three types — side by side:**

| | Point-to-Point VPN | Point-to-Site VPN | Site-to-Site VPN |
|---|---|---|---|
| **Connects** | One device/site ↔ one destination | One device ↔ one network | Entire network ↔ entire network |
| **Who initiates** | Client software or gateway | Client software (per device) | Gateway devices (automatic) |
| **Client software needed?** | Yes (on the single device) | Yes (on every remote device) | No — gateway handles it for all hosts |
| **Scale** | One fixed relationship | Many users, one site | Many sites, many networks |
| **AWS equivalent** | AWS Client VPN (one endpoint) | AWS Client VPN | AWS Site-to-Site VPN |
| **Typical user** | Remote employee, single branch | Remote employees, contractors | Whole offices, data centers |
| **Routing complexity** | Minimal | Moderate (split tunnel rules, auth rules) | High (route tables, BGP across sites) |
| **Authentication** | PSK / cert / user+pass | Certificate + MFA (user-centric) | PSK / cert (device-centric, no per-user auth) |
| **Connection persistence** | On-demand or persistent | On-demand (user initiates) | Always-on (gateway keeps tunnel up) |

---

## 6. Why use a VPN over a public, unencrypted connection

- **No new physical circuit** — runs over your existing internet connection.
- **Encryption in transit** — meets most compliance baselines for data crossing a public network.
- **Private IP addressing end-to-end** — internal hosts never need public IPs or exposed ports.
- **Fast to provision** — hours, not the weeks a dedicated line (like AWS Direct Connect) can take.
- **Lower cost** than dedicated circuits, for moderate and non-latency-critical throughput.

---

## 7. AWS services involved

| Service | What it's for |
|---|---|
| **AWS Site-to-Site VPN** | The core managed IPsec VPN service — connects an on-prem network (or another cloud) to a VPC or Transit Gateway. |
| **AWS Client VPN** | The Point-to-Site service — lets individual users' devices (laptops, phones) connect securely into a VPC using an OpenVPN-compatible client. Supports certificate, Active Directory, and SAML/MFA authentication. |
| **Virtual Private Gateway (VGW)** | The original AWS-side VPN endpoint, attached to a single VPC. |
| **Transit Gateway (TGW)** | A newer, more scalable AWS-side endpoint — terminates VPN connections and fans them out to many attached VPCs, other VPNs, and Direct Connect at once. |
| **AWS Direct Connect** | Not a VPN — a dedicated private physical circuit. Often combined *with* Site-to-Site VPN (as a backup path, or to encrypt traffic over DX itself via VPN-over-DX). |
| **CloudWatch** | Monitors tunnel status (UP/DOWN), data in/out, and tunnel state changes — used for alerting on tunnel failover. |

---

## 8. Components of an AWS Site-to-Site VPN connection

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

## 9. How it fits together (architecture)

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

## 10. Billing variables — what actually costs money

This is the part people miss until the bill arrives. Breaking it down:

| Cost component | How it's billed | Notes |
|---|---|---|
| **VPN Connection — hourly charge** | Per VPN connection, per hour it exists (whether or not it's passing traffic) | Charged from the moment the connection is created until deleted, regardless of tunnel UP/DOWN state. |
| **Data processed through the VPN connection** | Per GB, for data going *through* the VPN connection (in and out) | This is on top of the hourly charge — check current AWS pricing page for the per-GB rate in your region. |
| **Transit Gateway — attachment hourly charge** | Per TGW attachment, per hour (if using TGW instead of VGW) | A VPN attachment to a TGW is billed separately from the VPN connection's own hourly charge — this stacks. |
| **Transit Gateway — data processing charge** | Per GB processed through the TGW | Separate from the VPN's own per-GB charge if you route through TGW — the same byte can be billed once for VPN processing and again for TGW processing. |
| **Data transfer OUT to the internet** | Standard AWS data-transfer-out rates | Only applies where relevant — the VPN tunnel traffic itself isn't "internet egress" in the traditional sense, but check your architecture for any double-hop scenarios. |
| **VGW** | No separate hourly charge for the VGW resource itself (unlike TGW) | The VPN connection's hourly rate is the main cost when using VGW. |
| **Client VPN (Point-to-Site) — endpoint association hourly charge** | Per Client VPN endpoint subnet association, per hour | Each subnet you associate the endpoint with costs an hourly fee — independent of how many users are connected. |
| **Client VPN (Point-to-Site) — active connection hourly charge** | Per active client connection, per hour | A second, separate hourly charge per connected user — the more simultaneous remote workers, the higher this cost. |
| **CloudWatch charges** | Standard CloudWatch metrics/alarms pricing | Small, but adds up if you're polling tunnel state frequently or storing custom metrics. |

**Practical implication:** a VPN connection you forgot to delete after a test still bills hourly, even with zero traffic. Always check `describe-vpn-connections` before assuming cost is zero.

---

## 11. Advantages

- **Fast to provision** — a working tunnel in hours, no physical circuit needed.
- **Encrypted by default** — IPsec protects data in transit without extra effort.
- **Redundant by design** — two tunnels across two AZs, out of the box, for every connection.
- **Cost-effective** for low-to-moderate, non-latency-sensitive throughput compared to Direct Connect.
- **No new hardware in AWS** — the AWS side is fully managed (VGW/TGW); you only manage your on-prem router config.
- **Works over any existing internet connection** — no ISP coordination required beyond what you already have.
- **BGP support** gives automatic failover between tunnels without manual scripting.

## 12. Disadvantages

- **Throughput ceiling** — each individual tunnel is capped (historically around 1.25 Gbps per tunnel); high-throughput workloads need Direct Connect, ECMP across multiple VPN connections, or ECMP with TGW.
- **Variable latency and jitter** — it's still riding the public internet, so performance isn't guaranteed like a dedicated circuit.
- **Depends on internet reachability** — an ISP outage on either side takes the tunnel down; there's no SLA on the underlying internet path itself.
- **On-prem router compatibility matters** — older or non-standard routers may not support the exact IKE/IPsec parameters AWS requires, causing tunnel negotiation failures.
- **Static routing doesn't fail over automatically** — without BGP, a tunnel outage requires manual intervention or scripted route changes.
- **Costs stack in TGW architectures** — VPN hourly + TGW attachment hourly + two layers of per-GB processing can surprise teams used to VGW's simpler billing.
- **Not ideal for latency-sensitive or very high-bandwidth production workloads** — those belong on Direct Connect.

---



