# Amazon Route 53 — Complete Guide

## 1. What is Route 53?

Route 53 is Amazon's **DNS (Domain Name System) service**. In simple words — it's the "phonebook of the internet" for AWS.

When you type a website name like `www.amazon.com` in your browser, your computer doesn't understand "names" — it only understands **IP addresses** (like `54.239.28.85`). Route 53 translates the name into the IP address so your browser can connect to the right server.

**Why the name "53"?** Because DNS services run on **port 53**. Simple as that.

Route 53 also does two more jobs besides DNS:

- **Domain registration** — you can buy a domain (like `mywebsite.com`) directly from AWS.
- **Health checking** — it can check if your server is alive and route traffic away from a dead server.

### Easy Example

Imagine you save your friend's phone number as "Rahul" in your contacts. When you tap "Rahul," your phone doesn't call the word "Rahul" — it looks up the number and dials that. Route 53 is exactly this "contact list," but for websites.

---

## 2. Why use Route 53?

| Reason | Explanation |
| --- | --- |
| **Reliability** | AWS claims 100% availability SLA — very rare for any service |
| **Speed** | Uses a global network of DNS servers, so lookups are fast worldwide |
| **Scalable** | Handles a few requests or billions, no manual setup needed |
| **Integrates with AWS** | Works directly with EC2, S3, CloudFront, ELB, etc. — no extra config |
| **Smart routing** | Can send users to the "best" server based on location, load, or health |
| **Health checks** | Automatically stops sending traffic to a broken server |

### Real-world scenario

Suppose your company BEL has a website hosted on two EC2 servers — one in Mumbai (ap-south-1) and one in Singapore (ap-southeast-1). Route 53 can automatically send:

- Indian users → Mumbai server (faster, closer)
- Other Asian users → Singapore server

If the Mumbai server goes down, Route 53 detects it (via health check) and sends **everyone** to Singapore — without you doing anything manually.

---

## 3. Basic DNS Terms You Should Know First

| Term | Meaning | Example |
| --- | --- | --- |
| **Domain Name** | The website address | `example.com` |
| **Hosted Zone** | A container in Route 53 that holds all DNS records for a domain | Zone for `example.com` |
| **Record** | A single rule that says "this name points to this address" | `www.example.com → 1.2.3.4` |
| **TTL (Time To Live)** | How long a DNS answer is "remembered" (cached) before checking again | 300 seconds |
| **Name Servers (NS)** | Servers that actually answer DNS questions for your domain | AWS-assigned NS servers |

---

## 4. Hosted Zones (Very Important Concept)

A **Hosted Zone** is like a **folder** that stores all the DNS records for one domain.

Two types:

1. **Public Hosted Zone** — controls how traffic is routed on the **internet**. Example: `mycompany.com` is public so anyone in the world can reach it.
2. **Private Hosted Zone** — controls routing **inside your VPC only**. Example: `internal.mycompany.com` used only by your company's internal servers, not visible to the public internet.

### Easy Example

- Public Hosted Zone = your shop's signboard on the main road — anyone passing by can see it.
- Private Hosted Zone = a notice board inside your office — only employees (inside the VPC) can see it.

---

## 5. Record Types (Components of a Hosted Zone)

These are the actual "rules" inside a hosted zone.

| Record Type | Purpose | Example |
| --- | --- | --- |
| **A** | Maps a name to an IPv4 address | `example.com → 192.0.2.1` |
| **AAAA** | Maps a name to an IPv6 address | `example.com → 2001:db8::1` |
| **CNAME** | Maps a name to another name (not IP) | `blog.example.com → myblog.wordpress.com` |
| **Alias** (AWS-special) | Like CNAME but works even at the root domain (`example.com`) and points directly to AWS resources for free | `example.com → my-loadbalancer.aws.com` |
| **MX** | Directs email for the domain | `example.com → mail server` |
| **TXT** | Stores text info, often for verification (SPF, domain ownership, etc.) | Google Search Console verification |
| **NS** | Lists the name servers responsible for the domain | AWS name servers |
| **SOA** | Start of Authority — admin info about the zone (auto-created) | Zone metadata |
| **SRV** | Defines location of specific services (like a chat server) | Used in VoIP, gaming |
| **PTR** | Reverse of A record — IP to name (used for reverse DNS) | `192.0.2.1 → example.com` |

### Important AWS-only trick: Alias vs CNAME

- CNAME **cannot** be used on the root domain (`example.com`), only on subdomains (`www.example.com`).
- Alias record **can** be used on root domain, and it's **free** (no charge for Alias DNS queries to AWS resources), and it automatically updates if the target IP changes (like an ELB).

**Example**: You want `example.com` (no "www") to point to a Load Balancer.

- ❌ CNAME won't work here (DNS rules forbid it at root).
- ✅ Alias record works perfectly.

---

## 6. Routing Policies (Types of Routing) — The Core Feature

This is the **most important and most asked** part of Route 53. Routing policy = the *rule* Route 53 uses to decide **which server/IP to send the user to**.

### 6.1 Simple Routing

Sends all traffic to **one resource**. No fancy logic.

**Example**: `example.com → 3.3.3.3` — everyone gets sent to this one server. Best for a single-server website with no failover needed.

### 6.2 Weighted Routing

Splits traffic between multiple resources based on a **weight (percentage)** you assign.

**Example**: You're testing a new website version.

- Old version (Server A) → weight 90
- New version (Server B) → weight 10

Result: 90% of users go to A, 10% go to B. Great for **A/B testing** or **gradual rollouts**.

### 6.3 Latency-Based Routing

Sends the user to the AWS region that gives them the **fastest response time** (lowest latency) — not necessarily the closest by distance.

**Example**: A user in Dubai might get routed to your Mumbai server (low latency) even though Singapore is technically also close, because Mumbai responds faster for that user.

### 6.4 Geolocation Routing

Routes traffic based on the **user's physical location** (country/continent), regardless of speed.

**Example**:

- Users from India → Indian server (to comply with data laws, or show content in Hindi)
- Users from USA → US server (English content, US pricing)

Difference from Latency routing: Geolocation is about **compliance/content control**, Latency is about **speed**.

### 6.5 Geoproximity Routing (needs Traffic Flow)

Routes based on geographic location of users **and** resources, and lets you **"bias"** traffic — shift more or less traffic toward a resource by expanding or shrinking its geographic "pull area."

**Example**: You want your Singapore server to handle more of Southeast Asia even though Mumbai is technically closer for some users — you increase Singapore's bias value, and it "pulls" more of the traffic zone toward itself.

### 6.6 Failover Routing

Used for **active-passive** setup — one primary server, one backup. If the primary fails a health check, traffic automatically shifts to the backup.

**Example**: Your main website runs on EC2 in Mumbai (Primary). A backup static "Sorry, we're down" page sits on S3 (Secondary). If Mumbai server fails, Route 53 automatically serves the S3 backup page.

### 6.7 Multivalue Answer Routing

Returns **multiple healthy IP addresses** (up to 8) in response to a DNS query, and the client picks one — similar to basic load balancing but done at DNS level.

**Example**: You have 4 EC2 servers all running the same app. Multivalue routing returns all 4 healthy IPs, and each user's device randomly picks one to connect to. Not a replacement for a real load balancer, but useful for simple, cost-free load spreading.

### 6.8 IP-based Routing

Routes traffic based on the **client's IP address ranges (CIDR blocks)** that you define, instead of geography, latency, or health.

**Example**: You know a specific corporate office's public IP range (say `203.0.113.0/24`) always belongs to a partner company that should hit a dedicated server. You create a CIDR collection with that IP range and map it to that server — regardless of where the office is physically located. Useful when routing decisions depend on *who* the traffic belongs to (ISP, partner network) rather than *where* it's coming from geographically.

---

## 7. Quick Comparison Table of Routing Policies

| Policy | Based On | Best Use Case |
| --- | --- | --- |
| Simple | Nothing (1 resource) | Single small website |
| Weighted | % you set | A/B testing, canary deployment |
| Latency-based | Fastest response time | Global app needing speed |
| Geolocation | User's country/location | Legal compliance, localized content |
| Geoproximity | Location + bias | Fine control over regional traffic |
| Failover | Health check status | Disaster recovery, backup site |
| Multivalue Answer | Multiple healthy IPs | Simple load distribution |
| IP-based | Client's IP/CIDR range | Routing known networks (ISP, partner) to specific endpoints |

### Traffic Flow (visual policy builder)

When routing logic gets complex — e.g., "failover between two regions, but within each region split traffic by weight" — you don't have to build this by hand with nested records. **Traffic Flow** is a visual editor where you drag and connect routing rules into a tree, save it as a reusable **traffic policy**, and apply that same policy to multiple hosted zones at once. Useful when the same complex routing logic needs to be reused across several domains.

---

## 8. Health Checks

Route 53 can regularly "ping" (check) your server (every 10 or 30 seconds) to see if it's alive.

- If the server fails checks (e.g., 3 times in a row), Route 53 marks it **unhealthy**.
- Traffic is automatically routed away from unhealthy resources (used with Failover, Weighted, and Multivalue routing).

**Example**: Health check hits `http://yourserver.com/health`. If it doesn't get a `200 OK` response within the threshold, Route 53 considers it down and stops sending users there.

---

## 9. Domain Registration & Transfer

Route 53 also lets you **buy and manage domain names** directly (like `.com`, `.in`, `.org`), the same way you'd buy from GoDaddy — but it's all inside AWS, so DNS setup is automatic once you register.

You can also **transfer a domain**:

- **Transfer IN** — move a domain you own from another registrar (GoDaddy, Namecheap, etc.) into Route 53.
- **Transfer OUT** — move a domain from Route 53 to another registrar.

**Example**: Your company registered `bel-products.com` on GoDaddy years ago. You can transfer it into Route 53 so all DNS management sits in one place with the rest of your AWS infrastructure.

---

## 10. Route 53 Resolver (Hybrid DNS)

This is a **separate component** from hosted zones and records — it handles DNS resolution **between your VPC and an on-premises network** (or another VPC). Critical for hybrid/on-prem setups.

- **Inbound Endpoint** — lets DNS queries from **outside AWS** (your office network, data center) resolve names that live **inside your VPC**.
- **Outbound Endpoint** — lets resources **inside your VPC** resolve DNS names that live in your **on-premises network** (via VPN or Direct Connect).
- **Resolver Rules** — define which domain queries get forwarded where (e.g., "any query for `*.corp.local` goes to the on-prem DNS server at `10.0.5.5`").

### Easy Example

Your company BEL has:

- An on-prem data center with internal server `printer.bel.local`
- An AWS VPC running EC2 instances

Without Resolver, EC2 instances in the VPC can't resolve `printer.bel.local`, and on-prem machines can't resolve names inside the VPC. Route 53 Resolver bridges this gap — like a translator standing between two offices that speak different internal "languages," letting each side look up names on the other side.

This is a core piece for **hybrid network architecture** — VPC ↔ on-premises DNS resolution.

---

## 11. DNSSEC (DNS Security Extensions)

Route 53 supports **DNSSEC signing** for public hosted zones. This cryptographically signs your DNS records so resolvers can verify the response actually came from you and wasn't tampered with in transit.

**Why it matters**: Without DNSSEC, an attacker could intercept a DNS query and send back a fake IP address (DNS spoofing / cache poisoning), silently redirecting users to a malicious site. DNSSEC lets the resolver verify the signature and reject tampered answers.

**Example**: A bank's website enables DNSSEC so that if someone tries to redirect `bank.com` traffic to a fake server via a poisoned DNS cache, the resolver detects the signature mismatch and refuses to resolve it.

---

## 12. Query Logging

Route 53 can log every DNS query made against your public hosted zone to **CloudWatch Logs** (or S3/Kinesis Data Firehose).

**Useful for**:

- Auditing — who queried what, when
- Troubleshooting — confirming whether a query is even reaching Route 53
- Security — spotting unusual query patterns

**Example**: If users report "the site isn't loading," query logs let you confirm whether their DNS queries actually reached Route 53 and what answer was returned — narrowing down whether the issue is DNS or something downstream (server, network).

---

## 13. Pricing (High-Level — Always Check AWS Pricing Page for Current Rates)

| Item | Charged For |
| --- | --- |
| Hosted Zone | Flat monthly fee per hosted zone |
| DNS Queries | Per-million-queries fee (Alias queries to AWS resources like ELB/CloudFront/S3 are free) |
| Health Checks | Per health check per month (higher cost for checks with extra features, e.g., checking via multiple regions) |
| Domain Registration | Annual fee, varies by TLD (`.com`, `.in`, `.org`, etc.) |
| Resolver Endpoints | Per-endpoint hourly charge + per-query charge for Resolver queries |
| Traffic Flow | Small monthly fee per traffic policy record |

**Note**: Pricing changes over time and varies by region — confirm exact figures on the official AWS Route 53 pricing page before using in cost estimates.

---

## 14. Summary — All Components in One Place

```
Route 53
│
├── Domain Registration & Transfer   → Buy / move domain names
├── DNS Management
│   ├── Hosted Zones
│   │   ├── Public Hosted Zone   → Internet-facing
│   │   └── Private Hosted Zone  → Inside VPC only
│   └── Records
│       ├── A / AAAA / CNAME / Alias / MX / TXT / NS / SOA / SRV / PTR
├── Routing Policies
│   ├── Simple
│   ├── Weighted
│   ├── Latency-based
│   ├── Geolocation
│   ├── Geoproximity
│   ├── Failover
│   ├── Multivalue Answer
│   ├── IP-based
│   └── Traffic Flow (visual builder for complex combos)
├── Health Checks               → Monitor server status, auto-reroute
├── Route 53 Resolver           → Hybrid DNS (VPC ↔ on-premises)
├── DNSSEC                      → Cryptographic signing, prevents spoofing
└── Query Logging               → Logs DNS queries to CloudWatch/S3
```

---

## 15. One-Line Analogy to Remember Everything

> Route 53 = A super-smart phonebook (DNS) that not only tells you the number (IP) to call, but also decides **which** number to give you based on speed, location, health, or a percentage split you choose — and it can sell you the phone number (domain) too.
