# AWS Transit Gateway (TGW) — Simple Notes

## 1. What is TGW? (In One Line)

**TGW is like a central hub that connects all your VPCs together, instead of connecting each VPC to every other VPC one by one.**

Think of an airport with direct flights vs. a hub airport:

```
WITHOUT TGW (direct connections)          WITH TGW (hub)

VPC-A ---- VPC-B                                TGW
  |    \  /    |                              /  |  \
  |     \/     |                            VPC-A VPC-B VPC-C
  |     /\     |
  |    /  \    |
VPC-C ---- VPC-D
```

Every VPC just connects to TGW once, and TGW handles the rest.

---

## 2. Why Not Just Use VPC Peering?

VPC Peering = a direct cable between two VPCs. It's simple but doesn't scale.

**The math problem:** If you have `N` VPCs and want every one connected to every other one, you need:

```
N × (N - 1) / 2   connections
```

| VPCs | Peering Connections Needed |
|------|-----------------------------|
| 4    | 6                           |
| 10   | 45                          |
| 20   | 190                         |

That's a lot of cables to manage. Also, VPC Peering has one big limitation:

> ❌ **No transitive routing.** If A is peered with B, and B is peered with C, A **cannot** talk to C through B. You'd need a *direct* A–C peering too.

TGW fixes both problems: one connection per VPC, and traffic *can* pass through it to reach others.

---

## 3. All TGW Components & What Each One Is For

Here's a full reference table of every TGW component, in plain words, with why it exists.

| Component | Easy Meaning | Why / When You Use It |
|---|---|---|
| **Transit Gateway (TGW)** | The central hub/router itself | The core resource — everything else connects to or lives inside this |
| **Attachment** | The "doorway" linking a network to TGW | Needed anytime you want a VPC, VPN, on-prem link, another TGW, or an appliance to talk to TGW |
| **VPC Attachment** | Connects a single VPC to TGW | Standard way to plug a VPC into the hub |
| **VPN Attachment** | Connects a Site-to-Site VPN to TGW | Lets an on-prem network reach TGW over the internet (encrypted tunnel) |
| **Direct Connect (DX) Gateway Attachment** | Connects AWS Direct Connect to TGW | Lets on-prem reach TGW over a private, dedicated line (not the internet) |
| **TGW Peering Attachment** | Connects two separate TGWs together | Used when your TGWs are in different regions (or different accounts) and need to talk |
| **Connect Attachment** | Connects SD-WAN/appliances via GRE + BGP | Used for advanced integrations with third-party network appliances |
| **TGW Route Table** | The map TGW uses to decide where traffic goes | Every attachment's traffic gets routed by looking something up in one of these |
| **Association** | Tells an attachment *which* route table to USE | Required — without it, an attachment's traffic has no map to follow |
| **Propagation** | Tells a route table to automatically LEARN routes from an attachment | Saves you from manually typing every route by hand |
| **Static Route** | A route you type in manually | Used when you want full manual control, or propagation isn't available/desired |
| **Dynamic / BGP Routing** | Routes exchanged automatically via BGP | Used mainly with VPN/DX/Connect attachments to on-prem, so routes update themselves |
| **ASN (Autonomous System Number)** | The "ID badge" of a network in BGP | Needed on both sides of a BGP session so each network can identify itself |
| **Blackhole Route** | A route that silently drops matching traffic | Used to intentionally block a destination, or appears automatically if a route's target attachment is removed |
| **Route Selection (Longest Prefix Match)** | The rule TGW uses when multiple routes match | Not something you configure — just know TGW always picks the *most specific* matching route |
| **AWS RAM (Resource Access Manager)** | Shares the TGW across multiple AWS accounts | Used in multi-account setups so one central account can own the TGW and share it with others |
| **TGW Peering** | The connection type linking two TGWs | Used for cross-region (or cross-account) TGW-to-TGW connectivity |
| **TGW Connect** | GRE + BGP based attachment for appliances/SD-WAN | Used to integrate third-party network appliances or SD-WAN routers with TGW |
| **GRE Tunnel** | Wraps one packet inside another to form a tunnel | The transport mechanism underneath TGW Connect (no encryption by itself) |
| **Appliance Mode** | Keeps traffic flowing symmetrically through a stateful device | Turn this on when routing traffic through a firewall/inspection VPC, so replies take the same path as requests |
| **TGW CIDR Block** | A special CIDR assigned to TGW itself | Only relevant for TGW Connect / GRE addressing — not used like a normal VPC CIDR |
| **Multicast Support** | Lets TGW forward multicast traffic between VPCs | Used for applications that need multicast (e.g., media streaming, some financial apps) |
| **Quotas / Limits** | AWS-set limits (e.g., attachments per TGW, routes per table) | Good to know when designing large networks — check current AWS quota docs, since these can change and can often be raised via support request |

> 🧠 **Quick way to group these in your head:**
> - **Connectivity pieces** → Attachment, VPC/VPN/DX/Peering/Connect Attachment
> - **Routing pieces** → Route Table, Association, Propagation, Static Route, BGP, Blackhole, Route Selection
> - **Scaling/sharing pieces** → RAM, TGW Peering
> - **Special-purpose pieces** → Appliance Mode, TGW Connect, GRE, TGW CIDR, Multicast

---

## 4. The 5 Words You MUST Know

These five ideas are 90% of understanding TGW. Learn them first, everything else builds on top.

| Word | Easy Meaning |
|------|--------------|
| **TGW** | The central hub/router |
| **Attachment** | The "doorway" connecting a VPC (or VPN, etc.) to TGW |
| **Association** | *"Which route table should I USE?"* |
| **Propagation** | *"Which routes should I LEARN?"* |
| **TGW Route Table** | The map TGW uses to decide where to send traffic |

---

## 5. Attachment = The Doorway

An **attachment** is simply the connection between something (a VPC, a VPN, another TGW, etc.) and the TGW.

```
VPC ---- Attachment ---- TGW
```

Types of attachments:
- VPC Attachment
- VPN Attachment
- Direct Connect Attachment
- TGW Peering Attachment (connects two TGWs)
- Connect Attachment (for SD-WAN/appliances)

**Easy way to remember:** No attachment = no door = no traffic can enter or leave TGW through that network.

---

## 6. Association = "USE" 🔑

**Association** tells an attachment: *"When your traffic enters TGW, use THIS route table to decide where it goes."*

### Example
```
UAT Attachment  →  (Association)  →  UAT TGW Route Table
```
This means: "Traffic coming from the UAT VPC will look up its destination in the UAT TGW Route Table."

> 🧠 **Memory trick:** Association = USE (one attachment uses one route table)

---

## 7. Propagation = "LEARN" 🔑

**Propagation** tells a route table: *"Automatically learn the routes from this attachment."*

### Example
```
PROD Attachment  →  (Propagation)  →  TGW Route Table
```
Result: the TGW Route Table now knows:
```
10.90.0.0/16 → PROD Attachment
```

> 🧠 **Memory trick:** Propagation = LEARN (a route table learns routes from an attachment)

### ⚠️ Important gotcha
Creating an attachment does **NOT automatically mean** every route table learns its routes. Propagation has to be turned on for that specific pairing. Always double-check this — it's a common real-world mistake.

---

## 8. Association vs Propagation — Side by Side

This confuses almost everyone at first, so here's the simplest way to separate them:

| | Association | Propagation |
|---|---|---|
| **Question it answers** | "Which table do I use?" | "Which routes do I learn?" |
| **Direction** | Attachment → picks a route table | Attachment → sends its routes into a table |
| **Per attachment** | Only **one** route table can be associated | Can propagate into **many** route tables |

**Real analogy:** Imagine a school.
- **Association** = which classroom you are *assigned to* (only one).
- **Propagation** = which classrooms get *notified* about your test scores (could be several).

---

## 9. Full Example: UAT talking to PROD

Let's say:
- UAT VPC = `10.110.0.0/16`
- PROD VPC = `10.90.0.0/16`
- An EC2 in UAT (`10.110.1.10`) wants to reach an EC2 in PROD (`10.90.1.10`)

**Step-by-step journey:**

```
1. EC2-UAT (10.110.1.10) sends a packet to 10.90.1.10
                ↓
2. UAT VPC Route Table checks: "10.90.0.0/16 → send to TGW"
                ↓
3. Packet enters TGW through the UAT Attachment
                ↓
4. ASSOCIATION says: "UAT Attachment uses the UAT TGW Route Table"
                ↓
5. TGW looks up 10.90.0.0/16 in that table → "→ PROD Attachment"
   (this route was learned via PROPAGATION from the PROD attachment,
    or added manually as a static route)
                ↓
6. Packet exits through PROD Attachment
                ↓
7. PROD VPC Route Table delivers it to EC2-PROD (10.90.1.10)
```

**The one thing to remember:** the VPC route table must *first* point traffic to TGW — if it doesn't, the packet never even reaches TGW.

---

## 10. Static Route vs Propagation

Both end up putting an entry in the TGW Route Table — just two different ways to get there:

| | How it's added | Effort |
|---|---|---|
| **Propagation** | Automatic, if turned on | Low — TGW learns it for you |
| **Static Route** | You manually type it in | Manual, but gives you full control |

Example of a static route:
```
Destination: 10.90.0.0/16   →   Target: PROD Attachment
```
You're literally telling TGW: "for this destination, always send it here" — no auto-learning involved.

---

## 11. Blackhole Route = "Drop It" 🚫

A **blackhole route** tells TGW: *"If traffic matches this, don't send it anywhere — just delete it."*

```
Destination: 10.192.0.0/16  →  Target: BLACKHOLE
```

Any traffic to `10.192.x.x` simply disappears. No error, no forwarding — gone.

**When does this happen?**
- You configure it on purpose (e.g., to block a network)
- Or automatically, if the original target attachment gets deleted and the route has nowhere to go

---

## 12. Route Selection — "Most Specific Wins"

TGW uses **longest prefix match** — meaning, the *most specific* matching route always wins, just like normal VPC routing.

### Example
```
Routes in table:
10.0.0.0/8        (very broad)
10.90.0.0/16      (medium)
10.90.10.0/24     (very specific)

Traffic going to: 10.90.10.50
```
✅ TGW picks `10.90.10.0/24` because it's the **most specific** match — even though the other two routes technically also match.

---

## 13. Overlapping CIDRs — TGW Can't Fix This

**Important reality check:** TGW does **not** magically solve overlapping IP ranges.

```
VPC-A = 10.0.0.0/16
VPC-B = 10.0.0.0/16    ← SAME range as VPC-A!
```

If traffic is destined for `10.0.1.10`, TGW has **no way to know** if that means VPC-A or VPC-B. It's ambiguous — this setup simply won't work.

✅ **Best practice:** always give every VPC a unique CIDR range from the start:
```
VPC-A = 10.0.0.0/16
VPC-B = 10.1.0.0/16
VPC-C = 10.2.0.0/16
```

---

## 14. Connecting to On-Premises (VPN / Direct Connect)

TGW isn't just for VPCs — it can also be the hub for your on-premises data center.

```
On-Prem  ---(VPN or Direct Connect)---  TGW  --- VPC-PROD
```

This is powerful because now your on-prem network can reach *any* VPC attached to TGW, through one connection — not a separate VPN per VPC.

**BGP (Border Gateway Protocol)** is often used here so that routes are exchanged automatically instead of manually typed in:
- **BGP** = the "conversation" where networks tell each other what routes they have
- **ASN (Autonomous System Number)** = each network's unique "ID badge" in that conversation

You don't need deep BGP knowledge to get TGW basics — just know: *BGP = automatic route exchange, ASN = network's identity number.*

---

## 15. GRE Tunnel & TGW Connect (Advanced, Good to Know)

- **GRE (Generic Routing Encapsulation)** = wraps one packet inside another to create a "tunnel." It does **not** encrypt anything (that's a different job, done by IPsec).
- **TGW Connect** = a feature that uses GRE + BGP together to plug in SD-WAN devices or network appliances.

Easy way to remember:
```
GRE  = the tunnel (carries the traffic)
BGP  = the conversation (exchanges the routes)
```

---

## 16. TGW Peering — Connecting Two TGWs

If you have TGWs in **different AWS regions**, you can connect them directly:

```
Region A: TGW-1  <---- TGW Peering ---->  TGW-2 :Region B
```

This lets VPCs attached to TGW-1 talk to VPCs attached to TGW-2, across regions.

---

## 17. AWS RAM — Sharing TGW Across Accounts

**AWS RAM (Resource Access Manager)** lets one AWS account own the TGW, and *share* it with other accounts — very common in companies with multiple AWS accounts.

```
             Central Network Account
                      |
                     TGW
        /─────────────┼─────────────\
   Account-A       Account-B       Account-C
     VPC              VPC             VPC
```

This way, one team manages the network, while other teams just attach their VPCs to it.

---

## 18. Appliance Mode — For Firewalls

If traffic has to pass through a **firewall or security appliance**, that appliance often needs to see *both directions* of traffic (request AND reply) to work correctly.

**Appliance Mode** ensures traffic flows symmetrically — meaning it goes out and comes back through the same path, so the firewall doesn't get confused.

```
VPC-A → TGW → Firewall VPC → TGW → VPC-B
                (and the reply follows the SAME path back)
```

---

## 19. TGW vs VPC Peering — Quick Comparison

| Feature | VPC Peering | TGW |
|---|---|---|
| Connection style | Point-to-point (direct cable) | Hub-and-spoke |
| Transitive routing (A→B→C) | ❌ No | ✅ Yes |
| Good for | A handful of VPCs | Many VPCs, complex networks |
| VPN / on-prem as a hub | ❌ Not really | ✅ Yes |
| Hourly cost | None | Attachment + data processing fees |
| Best for | Simple, small setups | Large, scalable, centralized networks |

> ❗ Don't say "VPC Peering can't connect VPCs" — it can! It just can't do it *transitively* or *at scale* as easily as TGW.

---

## 20. Cost — Quick Note

TGW is **not free**. You generally pay for:
1. Each **attachment** (hourly)
2. **Data processing** (per GB through TGW)

VPC Peering has no hourly connection fee — just standard data transfer charges. So for a *simple* 2-VPC setup, peering might be cheaper. TGW's value comes from scale and centralized control, not from being the "cheap" option.

---

## 21. The Big Picture — Train Station Analogy 🚉

Think of TGW as a **central railway station**:

```
                    TGW = Central Station
                    /       |        \
                 UAT       PROD       DEV
                  |          |          |
                 VPC        VPC        VPC
                             |
                            VPN
                             |
                          On-Prem
```

- **Attachment** = the gate connecting a train line to the station
- **Association** = "which map does this gate use to find platforms?"
- **Propagation** = "which routes get announced to which maps?"
- **TGW Route Table** = the station's master map
- **Blackhole** = "this platform is closed — don't send anyone there"
- **BGP** = stations exchanging live updates about what routes they offer
- **Appliance Mode** = making sure trains take the same track both ways through a checkpoint

---

## 22. One-Line Answer (for interviews)

> AWS Transit Gateway is a centralized hub that connects multiple VPCs, VPNs, and on-prem networks. Traffic enters through an **attachment**, is looked up using the route table that attachment is **associated** with, and gets forwarded based on routes that were either **propagated** automatically or added as **static routes**.

---

## 23. The 5 Things to Never Forget

1. **TGW** = central hub
2. **Attachment** = the doorway in/out
3. **Association** = which table an attachment **USES**
4. **Propagation** = which routes a table **LEARNS**
5. **TGW Route Table** = the map deciding where traffic goes

**Full packet journey, memorized:**
```
EC2 → VPC Route Table → TGW Attachment → Association →
TGW Route Table → Route Lookup → Destination Attachment →
Destination VPC → Destination EC2
```
