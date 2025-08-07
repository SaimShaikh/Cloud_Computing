<img width="860" height="430" alt="image" src="https://github.com/user-attachments/assets/f2573201-ecf1-4e2b-8d6a-c69d4e80771b" />


# ⚖️ Load Balancers in AWS

AWS offers **Elastic Load Balancing (ELB)** to automatically distribute incoming application traffic across multiple targets like EC2 instances, containers, IP addresses, or Lambda functions. It improves:

- ✅ High availability
- ✅ Fault tolerance
- ✅ Scalability

---

## 📌 Types of Load Balancers

| Load Balancer Type           | Protocols Supported     | Best For                               | OSI Layer |
|------------------------------|-------------------------|----------------------------------------|-----------|
| **Application Load Balancer (ALB)** | HTTP, HTTPS              | Web apps, microservices (host/path-based routing) | Layer 7 |
| **Network Load Balancer (NLB)**     | TCP, UDP, TLS            | High-performance, low-latency apps     | Layer 4 |
| **Gateway Load Balancer (GWLB)**    | GENEVE (port 6081)       | 3rd-party appliances (firewalls, DPI)  | Layer 3 |
| **Classic Load Balancer (CLB)**     | HTTP, HTTPS, TCP, SSL    | Legacy applications                    | Layer 4 & 7 |

---

## 🔍 Load Balancer Details

### 🟩 Application Load Balancer (ALB)
- **Use Case**: Web apps, REST APIs, microservices
- **Features**:
  - Path-based and host-based routing
  - Native support for containers (ECS/EKS)
  - WebSocket support
  - Integrated authentication via Amazon Cognito or OIDC
  - HTTP/2 support

---

### 🟦 Network Load Balancer (NLB)
- **Use Case**: High-performance, low-latency applications (gaming, streaming, financial systems)
- **Features**:
  - Operates at Layer 4 (TCP/UDP/TLS)
  - Static IP and Elastic IP support
  - TLS offloading
  - Preserves source IP
  - Scales automatically to millions of requests per second

---

### 🟨 Gateway Load Balancer (GWLB)
- **Use Case**: Deploy, scale, and manage third-party virtual appliances like firewalls, deep packet inspection (DPI), and intrusion prevention systems (IPS)
- **Features**:
  - Operates at Layer 3
  - Uses GENEVE protocol
  - Transparent network traffic forwarding
  - Centralized management of security appliances

---

### 🟥 Classic Load Balancer (CLB)
- **Use Case**: Legacy applications (should not be used for new designs)
- **Features**:
  - Operates at both Layer 4 and Layer 7
  - Limited routing capabilities
  - No support for modern features like host/path-based routing
  - Less flexible compared to ALB/NLB

---

## 🧾 Comparison Table

| Feature                         | ALB              | NLB               | GWLB               | CLB               |
|----------------------------------|------------------|-------------------|--------------------|-------------------|
| OSI Layer                        | 7 (Application)  | 4 (Transport)     | 3 (Network)        | 4 & 7             |
| Protocols                        | HTTP, HTTPS      | TCP, UDP, TLS     | GENEVE             | HTTP, TCP, SSL    |
| Target Types                     | EC2, IP, Lambda  | EC2, IP           | Appliances         | EC2               |
| WebSocket Support                | ✅               | ❌                | ❌                 | ✅                |
| Path/Host-based Routing          | ✅               | ❌                | ❌                 | ❌                |
| Static IP Support                | ❌               | ✅                | ✅                 | ❌                |
| Container Support (ECS/EKS)      | ✅               | ✅                | ❌                 | ❌                |
| Use Case                         | Web, Microservices | High-performance workloads | Security appliances | Legacy apps       |

---

> ⚠️ **Note**: While Classic Load Balancer is still supported, it is **not recommended** for new deployments. Prefer ALB or NLB based on your use case.

