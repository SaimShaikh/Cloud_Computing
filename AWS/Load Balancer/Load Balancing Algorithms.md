# Load Balancing Algorithms — Complete Guide

> A single-file reference covering load balancing fundamentals, algorithms, AWS implementations, system design considerations, and interview preparation.

---

## Table of Contents

1. [Introduction to Load Balancing](#1-introduction-to-load-balancing)
2. [Why Load Balancing Is Required](#2-why-load-balancing-is-required)
3. [End-to-End Request Flow](#3-end-to-end-request-flow)
4. [OSI Layer: L4 vs L7 Load Balancing](#4-osi-layer-l4-vs-l7-load-balancing)
5. [Static vs Dynamic Load Balancing](#5-static-vs-dynamic-load-balancing)
6. [Core Load Balancing Algorithms](#6-core-load-balancing-algorithms)
   - [6.1 Round Robin](#61-round-robin)
   - [6.2 Weighted Round Robin](#62-weighted-round-robin)
   - [6.3 Source IP Hash](#63-source-ip-hash)
   - [6.4 Least Connections](#64-least-connections)
   - [6.5 Least Response Time](#65-least-response-time)
   - [6.6 Resource-Based Load Balancing](#66-resource-based-load-balancing)
7. [Additional Algorithms](#7-additional-algorithms)
8. [AWS Elastic Load Balancer (ALB, NLB, GWLB, CLB)](#8-aws-elastic-load-balancer-alb-nlb-gwlb-clb)
9. [AWS Architecture Diagrams](#9-aws-architecture-diagrams)
10. [System Design Considerations](#10-system-design-considerations)
11. [Advantages & Disadvantages](#11-advantages--disadvantages)
12. [Best Practices](#12-best-practices)
13. [Interview Questions](#13-interview-questions)
14. [Summary Table](#14-summary-table)
15. [Mastery Checklist](#15-mastery-checklist)

---

## 1. Introduction to Load Balancing

**Load balancing** is the process of distributing incoming network traffic or application requests across multiple backend servers (also called nodes, targets, or instances) so that no single server becomes a bottleneck, single point of failure, or performance chokepoint.

A load balancer sits logically between the client and the server fleet. It does not process business logic — it decides **which backend** should handle a given request/connection, based on an **algorithm**.

<img width="1402" height="1122" alt="image" src="https://github.com/user-attachments/assets/436d0b17-5d33-4e2a-8e18-d595c0c4d219" />


At its core, a load balancer performs three jobs:

1. **Health checking** — continuously verify backend servers are alive and able to serve traffic.
2. **Traffic distribution** — apply an algorithm to route each request/connection.
3. **Failure handling** — remove unhealthy targets from rotation and reintroduce them once healthy.

---

## 2. Why Load Balancing Is Required

| Problem Without Load Balancing | How Load Balancing Solves It |
|---|---|
| Single server = single point of failure | Traffic reroutes to healthy servers automatically |
| One server gets overwhelmed while others sit idle | Even/weighted distribution of requests |
| No way to scale horizontally | New servers can be added behind the LB transparently |
| Downtime during deployments | Rolling deployments — drain and replace targets one at a time |
| Poor global latency | LB + geo routing sends users to the nearest region |
| Manual failover | Health checks + automatic target removal |
| Security exposure (all servers public) | LB is the single public entry point; backends stay private |

**Real-world analogy:** Imagine a bank with one teller counter and 500 customers. A load balancer is the "next counter please" system — it opens more counters (servers) as needed and sends each customer (request) to whichever counter is free or best suited, instead of everyone crowding one window.

Key motivations, summarized:

- **High Availability (HA)** — no single server failure takes down the whole application.
- **Scalability** — horizontally scale by adding more backend instances.
- **Performance** — reduce latency by avoiding overloaded servers.
- **Fault tolerance** — unhealthy instances are automatically excluded.
- **Maintainability** — servers can be patched/replaced without downtime.

---

## 3. End-to-End Request Flow

```
 1. DNS Resolution
    Client resolves app.example.com → Load Balancer IP/DNS (e.g., ALB DNS name)

 2. TCP/TLS Handshake
    Client establishes connection with the Load Balancer (TLS termination may occur here)

 3. Listener Match
    LB checks listener rules (port 443, protocol HTTPS, path /api/*, host header, etc.)

 4. Routing Decision
    LB consults Target Group(s) + Load Balancing Algorithm to pick a backend

 5. Health Check Filter
    Only "healthy" targets are eligible candidates

 6. Request Forwarded
    LB opens/reuses a connection to the chosen backend and forwards the request

 7. Backend Processes Request
    App server executes business logic, queries DB/cache, etc.

 8. Response Returned
    Backend → LB → Client (LB may add headers like X-Forwarded-For)

 9. Connection Handling
    LB decides to keep-alive, close, or recycle the connection based on config
```

ASCII sequence view:

```
Client            DNS             Load Balancer         Target Group        Backend
  │                │                     │                    │                │
  │──resolve name─▶│                     │                    │                │
  │◀──LB IP────────│                     │                    │                │
  │──TCP/TLS connect──────────────────▶  │                    │                │
  │                │                     │──check health─────▶│                │
  │                │                     │◀──healthy list─────│                │
  │                │                     │──apply algorithm──▶│                │
  │                │                     │──forward request───────────────────▶│
  │                │                     │                    │◀──process──────│
  │◀────────────────────response─────────│                    │                │
```

---

## 4. OSI Layer: L4 vs L7 Load Balancing

Load balancers operate at different layers of the OSI model, and this fundamentally changes **what information** they can use to route traffic.

```
OSI Model                          Load Balancer Type
┌─────────────────────┐
│ L7 - Application     │◀──────────  Application Load Balancer (ALB)
├─────────────────────┤             (HTTP/HTTPS, headers, cookies, path, host)
│ L6 - Presentation    │
├─────────────────────┤
│ L5 - Session         │
├─────────────────────┤
│ L4 - Transport       │◀──────────  Network Load Balancer (NLB)
├─────────────────────┤             (TCP/UDP, IP + Port only)
│ L3 - Network         │◀──────────  Gateway Load Balancer (GWLB)
├─────────────────────┤             (IP packet-level, works with L3 GENEVE)
│ L2 - Data Link       │
├─────────────────────┤
│ L1 - Physical        │
└─────────────────────┘
```

### L4 Load Balancing (Transport Layer)

- Operates on **IP address + Port + Protocol** (TCP/UDP).
- Does **not** inspect packet content (no visibility into HTTP headers, cookies, URLs).
- Extremely fast, low-latency, high-throughput — it's essentially forwarding packets.
- Cannot make content-based routing decisions (e.g., route `/api` vs `/static` differently).

### L7 Load Balancing (Application Layer)

- Understands HTTP/HTTPS, gRPC, WebSockets.
- Can route based on: **URL path, hostname, headers, cookies, query strings, HTTP method**.
- Can perform TLS termination, request/response rewriting, authentication (OIDC/Cognito), and WAF integration.
- Slightly higher latency than L4 due to deeper packet inspection, but negligible for most use cases.

| Feature | L4 (NLB) | L7 (ALB) |
|---|---|---|
| Routing basis | IP + Port | URL, Host, Header, Cookie |
| Protocols | TCP, UDP, TLS | HTTP, HTTPS, gRPC, WebSocket |
| Speed | Extremely fast | Fast (slightly more overhead) |
| Content inspection | No | Yes |
| SSL Termination | Optional (TLS listener) | Yes, native |
| Sticky sessions | IP-based only | Cookie-based |
| Use case | Gaming, IoT, extreme low-latency TCP/UDP | Microservices, web apps, APIs |

---

## 5. Static vs Dynamic Load Balancing

### Static Load Balancing

The routing decision **does not consider real-time server state** (CPU, connections, response time). Distribution is decided by a fixed rule set upfront.

- Examples: **Round Robin, Weighted Round Robin, Source IP Hash, URL Hash**
- Pros: simple, predictable, low computational overhead on the LB
- Cons: can send traffic to an already-overloaded server if it "wins" the static rule, since server health/load isn't factored into the decision in real time

### Dynamic Load Balancing

The routing decision **adapts in real time** based on current server metrics.

- Examples: **Least Connections, Least Response Time, Resource-Based (adaptive) Load Balancing**
- Pros: more efficient distribution, adapts to uneven request costs
- Cons: requires the LB to continuously collect metrics (adds slight overhead/complexity)

```
STATIC                                   DYNAMIC
┌─────────────┐                         ┌─────────────┐
│ Fixed Rule  │                         │ Live Metrics │
│ (e.g. RR)   │                         │ (conns, CPU, │
│             │                         │  latency)    │
└──────┬──────┘                         └──────┬───────┘
       │                                       │
       ▼                                       ▼
 Request N goes to                     Request N goes to
 Server (N mod count)                  Server with best
                                        current metric
```

---

## 6. Core Load Balancing Algorithms

### 6.1 Round Robin

Round Robin is a simple static load balancing technique that distributes incoming requests to servers in a fixed sequential or rotational order. It is commonly used due to its ease of implementation.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/e7bda416-2c1c-4e45-a5e2-da7a03aab8dc" />


- **Best for:** Homogeneous servers (identical specs) with roughly equal-cost requests.
- **Weakness:** Ignores actual server load — a slow server keeps getting the same share as a fast one.

### 6.2 Weighted Round Robin

Weighted Round Robin is a static load balancing technique similar to Round Robin, but it distributes requests based on assigned weight values that represent each server’s capacity.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/ef2c7590-413b-4afe-b97d-381708d7b9b9" />


- **Best for:** Heterogeneous fleets (e.g., mix of large and small EC2 instance types).
- **Weakness:** Weights are usually static/manual — doesn't adapt to real-time load unless recalculated.

### 6.3 Source IP Hash

The Source IP Hash Load Balancing Algorithm is a static network routing technique that distributes incoming traffic across a pool of servers by generating a unique mathematical hash key from the client's source IP address. Because the calculation is deterministic, it ensures session persistence (server affinity), meaning a specific client will consistently be routed to the exact same backend server for the duration of their session.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/fbbe9277-333c-48b8-9bab-ffcd0cab84dd" />


- **Best for:** Session persistence ("sticky sessions") without needing cookies — useful for stateful apps, gaming servers, or protocols without a session layer.
- **Weakness:** Uneven distribution if client IPs aren't uniformly distributed (e.g., many users behind one corporate NAT IP all land on one server). Also, if a server is added/removed, hash results shift for many clients (unless consistent hashing is used — see Section 7).

### 6.4 Least Connections

The Least Connections algorithm is a dynamic load balancing technique that routes new requests to the server with the fewest active connections. It focuses on balancing workload by considering the current load on each server.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/a519eaf1-bc2a-4ee3-b952-d0f1c853fa1e" />


- **Best for:** Long-lived connections with variable duration (e.g., database proxies, WebSocket servers, persistent API sessions).
- **Weakness:** "Number of connections" isn't always a good proxy for "actual load" — a connection could be idle or extremely CPU-heavy.

### 6.5 Least Response Time

The Least Response method is a dynamic load balancing approach that aims to minimize response times by directing new requests to the server with the quickest response time. 



<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/93e105b2-00e7-4836-93bb-929b932501a3" />


- **Best for:** Latency-sensitive applications (real-time APIs, trading platforms, live user-facing requests).
- **Weakness:** More computationally expensive; requires continuous latency sampling; can flap between servers if latency is noisy.

### 6.6 Resource-Based Load Balancing

Resource-Based Load Balancing assigns incoming requests to servers based on their current resource availability, such as CPU usage, memory, or bandwidth, ensuring efficient and balanced system performance.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/db2ca759-2b55-4587-8547-98c89d9bab21" />


- **Best for:** Compute-heavy workloads (video transcoding, ML inference, batch processing) where CPU/memory is the real constraint, not just connection count.
- **Weakness:** Requires an agent or metrics-reporting mechanism on each backend; added architectural complexity.

---

## 7. Additional Algorithms

| Algorithm | How It Works | Typical Use Case |
|---|---|---|
| **Random** | Picks a server uniformly at random for each request | Simple, stateless workloads; surprisingly effective at scale with large N |
| **Weighted Least Connections** | Least Connections, but connections are divided by a server's weight/capacity before comparing | Heterogeneous fleets with variable connection load |
| **Consistent Hashing** | Maps both servers and request keys onto a hash ring; a request goes to the next server clockwise on the ring | Distributed caches (Memcached, Redis Cluster), CDN edge routing — minimizes re-mapping when servers are added/removed |
| **URL Hash** | Hashes the request URL to consistently route the same URL to the same backend | Caching proxies — maximizes cache hit rate per backend |
| **IP Hash (Layer 3 variant)** | Similar to Source IP Hash but purely at the network layer, sometimes combining src+dst | Network appliances, firewalls |
| **Least Bandwidth** | Routes to the server currently transmitting the least amount of data (Mbps) | Streaming/media delivery |
| **Power of Two Choices** | Randomly pick 2 servers, then route to whichever of the two has fewer active connections | Large-scale systems (avoids the overhead of tracking all servers while beating pure Random) |

**Consistent Hashing ring visualization:**

```
                    0°
                     │
        S3 ●─────────┼─────────● S1
                     │
      270°───────────┼─────────── 90°
                     │
        S2 ●─────────┼─────────
                     │
                    180°

Request key "user123" hashes to 95° → goes to next server clockwise → S2
Adding a new server S4 at 100° only remaps keys between 95°-100°,
NOT the entire ring (unlike modulo hashing).
```

---

## 8. AWS Elastic Load Balancer (ALB, NLB, GWLB, CLB)

AWS offers four types of managed load balancers under the **Elastic Load Balancing (ELB)** umbrella.

### 8.1 Application Load Balancer (ALB) — Layer 7

- Routes HTTP/HTTPS/gRPC/WebSocket traffic.
- **Routing basis:** Host header, path, HTTP method, query string, source IP, headers.
- Native integrations: **AWS WAF, Cognito authentication, Lambda targets, ECS/EKS service discovery**.
- Supports **Target Groups** with algorithm choice: Round Robin (default) or Least Outstanding Requests.

### 8.2 Network Load Balancer (NLB) — Layer 4

- Routes TCP/UDP/TLS traffic at extremely high throughput with ultra-low latency.
- Preserves the client's source IP (unless using a target group with proxy protocol).
- Supports **static IP / Elastic IP per Availability Zone** — useful for allow-listing.
- Ideal for: gaming backends, IoT, financial trading systems, or as a front door for third-party firewall appliances.

### 8.3 Gateway Load Balancer (GWLB) — Layer 3/4 (GENEVE)

- Designed to deploy, scale, and manage **third-party virtual appliances** (firewalls, IDS/IPS, deep packet inspection).
- Uses the **GENEVE protocol (port 6081)** to transparently pass traffic through appliances and back into the network path.
- Combines a transparent network gateway with load balancing across appliance instances.

### 8.4 Classic Load Balancer (CLB) — Legacy L4/L7

- AWS's original load balancer (pre-2016 architecture, EC2-Classic era).
- Supports basic L4 and limited L7 (HTTP) features.
- **AWS recommends migrating to ALB/NLB** — CLB lacks modern features like WAF integration, native gRPC, or advanced routing rules.

| Feature | ALB | NLB | GWLB | CLB |
|---|---|---|---|---|
| OSI Layer | L7 | L4 | L3/L4 | L4 / limited L7 |
| Protocols | HTTP, HTTPS, gRPC, WS | TCP, UDP, TLS | IP (GENEVE) | HTTP, HTTPS, TCP |
| Static IP | No (use NLB in front if needed) | Yes | Yes | No |
| Use case | Web apps, microservices | Extreme performance, gaming | Firewalls/appliances | Legacy apps |
| WAF support | Yes | No | No | No |
| Target types | Instance, IP, Lambda | Instance, IP, ALB | Appliance instance/IP | Instance |

---

## 9. AWS Architecture Diagrams

### 9.1 Basic ALB Multi-AZ Web Architecture

```
                              Route 53 (DNS)
                                    │
                                    ▼
                     ┌─────────────────────────────┐
                     │  Application Load Balancer   │
                     │        (Multi-AZ)            │
                     └───────────┬─────────────────┘
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
     ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
     │   AZ-a           │ │   AZ-b           │ │   AZ-c           │
     │ ┌─────────────┐  │ │ ┌─────────────┐  │ │ ┌─────────────┐  │
     │ │ Target Group │  │ │ │ Target Group │  │ │ │ Target Group │  │
     │ │  EC2 / ECS   │  │ │ │  EC2 / ECS   │  │ │ │  EC2 / ECS   │  │
     │ └─────────────┘  │ │ └─────────────┘  │ │ └─────────────┘  │
     └─────────────────┘ └─────────────────┘ └─────────────────┘
              │                   │                   │
              └───────────────────┼───────────────────┘
                                   ▼
                          ┌─────────────────┐
                          │   RDS (Multi-AZ) │
                          └─────────────────┘
```

### 9.2 NLB in Front of a Firewall Appliance (via GWLB)

```
     Internet
        │
        ▼
┌───────────────┐
│  NLB (public)  │
└───────┬───────┘
        ▼
┌────────────────────────┐
│  Gateway Load Balancer   │
│   Endpoint (GWLBe)       │
└───────────┬─────────────┘
            ▼
   ┌──────────────────┐
   │ 3rd-Party Firewall│
   │   Appliance Fleet  │
   └─────────┬─────────┘
             ▼
   ┌──────────────────┐
   │  Application VPC  │
   │  (private subnet) │
   └──────────────────┘
```

### 9.3 Path-Based Routing with ALB (Microservices)

```
                       ┌───────────────────────────┐
                       │           ALB              │
                       │  Listener: 443 (HTTPS)      │
                       └──────────────┬─────────────┘
        ┌───────────────────┬─────────┼─────────┬───────────────────┐
        ▼                   ▼                   ▼                   ▼
   /api/users*         /api/orders*        /static/*            /ws/*
        │                   │                   │                   │
        ▼                   ▼                   ▼                   ▼
 ┌──────────────┐    ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
 │ Users Target  │    │ Orders Target │   │  S3 / CDN     │   │ WebSocket TG  │
 │    Group      │    │    Group      │   │  (via ALB)    │   │  (sticky)     │
 └──────────────┘    └──────────────┘   └──────────────┘   └──────────────┘
```

### 9.4 Cross-Zone Load Balancing

```
Without Cross-Zone LB:                  With Cross-Zone LB Enabled:

AZ-a: LB node → only AZ-a targets       AZ-a: LB node → ALL targets in
      (2 targets)                              AZ-a AND AZ-b (evenly)
AZ-b: LB node → only AZ-b targets       AZ-b: LB node → ALL targets in
      (6 targets, overloaded!)                 AZ-a AND AZ-b (evenly)

Result: AZ-b targets overloaded         Result: Even load regardless of
        while AZ-a targets idle                 how targets are distributed
                                                 across AZs
```

---

## 10. System Design Considerations

When designing a load-balanced system, evaluate these dimensions:

1. **Statefulness of the application**
   - Stateless services → any algorithm works (Round Robin, Least Connections).
   - Stateful services (in-memory sessions) → need sticky sessions (cookie-based on ALB, or Source IP Hash on L4) or externalize session state (Redis/ElastiCache).

2. **Health check design**
   - Shallow health check (`/health` returns 200) vs deep health check (verifies DB connectivity, cache, downstream dependencies).
   - Tune `interval`, `timeout`, `healthy threshold`, `unhealthy threshold` to balance fast failure detection vs false positives.

3. **Connection draining / deregistration delay**
   - When removing a target (deploy, scale-in), allow in-flight requests to complete before terminating (AWS default: 300s deregistration delay).

4. **Multi-AZ / Multi-Region resilience**
   - Spread targets across AZs; use Route 53 with latency-based or failover routing for multi-region.

5. **Autoscaling integration**
   - LB target groups should be tied to an Auto Scaling Group so capacity grows/shrinks with demand, and the LB's algorithm naturally adapts to the new fleet size.

6. **TLS termination point**
   - Terminate at the LB (simpler cert management) vs end-to-end encryption (LB → backend also encrypted, required for strict compliance like PCI-DSS/HIPAA).

7. **Algorithm choice vs workload shape**
   - Uniform, short-lived requests → Round Robin / Weighted Round Robin.
   - Long-lived, variable-duration connections → Least Connections.
   - Latency-critical → Least Response Time.
   - CPU/memory-bound backends → Resource-Based.
   - Cache-friendly routing → Consistent Hashing / URL Hash.

8. **Observability**
   - Track per-target latency, 5xx rate, active connection count, and target health flapping (CloudWatch metrics: `TargetResponseTime`, `HealthyHostCount`, `UnHealthyHostCount`, `HTTPCode_Target_5XX_Count`).

9. **Failure domains**
   - Avoid a single LB being a new single point of failure — AWS ELB is inherently multi-AZ and horizontally scaled internally, but self-managed LBs (e.g., HAProxy/Nginx on EC2) need their own HA pair + Elastic IP failover or DNS failover.

10. **Cost model**
    - ALB/NLB pricing = hourly + **LCU (Load Balancer Capacity Unit)** based on new connections, active connections, processed bytes, and rule evaluations — factor this into cost estimates for high-throughput systems.

---

## 11. Advantages & Disadvantages

### Advantages

- Improves **availability** — no single server is a single point of failure.
- Enables **horizontal scalability** — add/remove servers transparently.
- Improves **performance** by avoiding overloaded nodes.
- Enables **zero-downtime deployments** (rolling/blue-green).
- Centralizes **TLS termination, WAF, and authentication**.
- Provides **observability** (centralized metrics/logs for all traffic).

### Disadvantages

- Adds an **additional network hop** (minor latency overhead).
- Becomes a **critical dependency** — if misconfigured, it can take down the entire application even with healthy backends.
- **Algorithm mismatch** can cause uneven load (e.g., Round Robin on heterogeneous servers).
- **Sticky sessions** reduce load-balancing effectiveness and can create hot spots.
- **Cost** — managed LBs (ALB/NLB) incur hourly + usage-based (LCU) charges.
- **Complexity** in multi-region/multi-cluster setups (health check propagation, DNS TTLs, failover timing).

---

## 12. Best Practices

1. **Always enable cross-zone load balancing** unless you have a specific reason not to (ALB enables it by default; NLB requires explicit enablement).
2. **Use deep health checks** that reflect real application readiness, not just process liveness.
3. **Set sensible deregistration delay** — long enough to drain in-flight requests, short enough not to slow down deployments.
4. **Avoid sticky sessions where possible** — externalize session state (Redis/DynamoDB) so any backend can serve any request.
5. **Match the algorithm to the workload** — don't default to Round Robin for heterogeneous fleets or long-lived connections.
6. **Terminate TLS at the LB** unless compliance mandates end-to-end encryption; if it does, use HTTPS/TLS listeners all the way to the target.
7. **Separate public and private load balancers** — internet-facing ALB in public subnets, internal-facing ALB/NLB in private subnets for service-to-service traffic.
8. **Automate target registration** via Auto Scaling Groups / ECS service integration — never manually add/remove targets in production.
9. **Monitor 5xx and latency metrics per target**, not just aggregate — a single bad target can be masked by aggregate averages.
10. **Use Weighted Target Groups for blue/green and canary deployments** (ALB supports weighted routing across multiple target groups on the same listener rule).
11. **Set connection idle timeout appropriately** — default ALB idle timeout is 60s; increase for long-polling/WebSocket use cases.
12. **Use WAF + ALB together** for L7 applications exposed to the internet.

---

## 13. Interview Questions

**Q1: What is the difference between Layer 4 and Layer 7 load balancing?**
> L4 routes based on IP/port/protocol without inspecting payload (fast, protocol-agnostic). L7 inspects the actual application-layer request (HTTP headers, path, cookies) to make smarter, content-aware routing decisions, at the cost of slightly more processing overhead.

**Q2: How does Round Robin differ from Weighted Round Robin?**
> Round Robin distributes requests equally and sequentially across all servers, assuming they have equal capacity. Weighted Round Robin assigns a capacity-proportional weight to each server so more capable servers receive a proportionally larger share of traffic.

**Q3: When would you choose Least Connections over Round Robin?**
> When requests have highly variable processing time or connection duration (e.g., persistent WebSocket sessions, database proxy connections) — Round Robin would blindly cycle through servers even if one is already handling many long-lived, expensive connections.

**Q4: What problem does Consistent Hashing solve that simple modulo hashing does not?**
> With simple modulo hashing (`hash(key) % N`), adding or removing a single server changes the modulo result for almost all keys, causing massive cache invalidation/remapping. Consistent Hashing places servers and keys on a hash ring so that only the keys adjacent to the changed server are remapped — everything else stays put.

**Q5: How does an ALB differ from an NLB in terms of source IP preservation?**
> NLB preserves the client's original source IP by default (operating at L4, it passes the packet through with minimal modification). ALB, since it terminates the TCP connection and re-establishes a new one to the backend, requires the `X-Forwarded-For` header to convey the original client IP.

**Q6: What is connection draining / deregistration delay, and why does it matter?**
> It's the grace period during which a load balancer stops sending *new* requests to a target being deregistered but allows *existing in-flight* requests to complete, preventing dropped connections during scale-in or deployment.

**Q7: Why might sticky sessions hurt load balancing effectiveness?**
> Because they pin a client to one server for the session's duration, some servers can become overloaded if the traffic pattern is uneven (e.g., a few "power users" or bots), defeating the purpose of even distribution. The recommended fix is externalizing session state instead.

**Q8: What's the difference between static and dynamic load balancing algorithms?**
> Static algorithms make routing decisions using a fixed rule that doesn't consider current server state (e.g., Round Robin, Hashing). Dynamic algorithms factor in real-time metrics like active connections, response time, or CPU/memory utilization to make adaptive decisions (e.g., Least Connections, Least Response Time, Resource-Based).

**Q9: How does a Gateway Load Balancer differ architecturally from ALB/NLB?**
> GWLB operates at L3/L4 using the GENEVE protocol to transparently insert third-party network appliances (firewalls, IDS/IPS) into the traffic path — it's designed for appliance load balancing/scaling, not for balancing application backend servers directly like ALB/NLB do.

**Q10: How would you design health checks to avoid false-positive failovers?**
> Use a threshold-based approach — require multiple consecutive failed checks (`unhealthy threshold`) before marking a target unhealthy, and multiple consecutive passes (`healthy threshold`) before marking it healthy again, combined with a reasonable check interval/timeout to avoid reacting to transient blips (e.g., a single slow GC pause).

**Q11: What is "Power of Two Choices" and why is it used at scale?**
> Instead of tracking global state for every server (expensive at massive scale) or picking purely at random (uneven), the LB randomly samples 2 servers and picks the one with fewer active connections. This gives load distribution close to full Least Connections at a fraction of the tracking overhead.

**Q12: In AWS, how do you achieve blue/green or canary deployments using a load balancer?**
> Use ALB with weighted target groups on the same listener rule — direct e.g. 95% of traffic to the "blue" (stable) target group and 5% to the "green" (canary) target group, then shift the weight gradually as confidence increases.

---

## 14. Summary Table

| Algorithm | Type | Considers Server State? | Best Use Case | Weakness |
|---|---|---|---|---|
| Round Robin | Static | No | Homogeneous servers, equal-cost requests | Ignores real load |
| Weighted Round Robin | Static | No (weights are pre-set) | Heterogeneous server capacities | Weights need manual tuning |
| Source IP Hash | Static | No | Session persistence without cookies | Uneven if IPs aren't diverse |
| Least Connections | Dynamic | Yes (connection count) | Long-lived / variable-duration connections | Connections ≠ true load |
| Least Response Time | Dynamic | Yes (latency + connections) | Latency-sensitive apps | More overhead, can flap |
| Resource-Based | Dynamic | Yes (CPU/memory/custom metrics) | CPU/memory-bound workloads | Needs agent/metrics reporting |
| Random | Static | No | Simple, large-scale stateless systems | No guarantee of even split short-term |
| Consistent Hashing | Static | No | Distributed caches, CDN routing | Slight complexity to implement |
| URL Hash | Static | No | Cache-friendly reverse proxies | Hotspot if one URL is very popular |
| Power of Two Choices | Hybrid | Partial (sampled) | Very large fleets | Slightly less optimal than full Least Conn |

---

## 15. Mastery Checklist

- [ ] Can explain why load balancing is necessary using availability, scalability, and performance arguments
- [ ] Can draw the end-to-end request flow from DNS resolution to backend response
- [ ] Can clearly differentiate L4 vs L7 load balancing with examples
- [ ] Can explain static vs dynamic algorithms and give one example of each
- [ ] Can walk through Round Robin, Weighted Round Robin, Source IP Hash, Least Connections, Least Response Time, and Resource-Based algorithms with a worked example for each
- [ ] Can explain Consistent Hashing and why it minimizes remapping vs modulo hashing
- [ ] Can map AWS ELB types (ALB/NLB/GWLB/CLB) to correct OSI layer and use case
- [ ] Can draw a multi-AZ ALB architecture diagram from memory
- [ ] Can explain cross-zone load balancing and its impact on distribution
- [ ] Can list at least 5 system design considerations (health checks, draining, statefulness, TLS termination, autoscaling integration)
- [ ] Can answer all 12 interview questions above without notes
- [ ] Can recommend the right algorithm given a workload description (interview-style scenario question)

---

*End of guide.*
