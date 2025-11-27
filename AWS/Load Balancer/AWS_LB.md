# AWS Load Balancers Explained 

A comprehensive guide to understanding Application (ALB), Network (NLB), Gateway (GWLB), and Classic (CLB) Load Balancers in AWS. This guide covers real-world scenarios, features, and when to use each type.

---

## 1. Application Load Balancer (ALB)
**Layer 7 (HTTP/HTTPS)**

### 🛒 The Scenario
Imagine you are building a **modern e-commerce microservices application**. 
*   Users need to browse products at `example.com/products` (handled by a Product Service container).
*   They need to checkout at `example.com/checkout` (handled by a Checkout Service container).
*   You need to block malicious SQL injection attacks using a Web Application Firewall (WAF).
*   You want users to log in using their Google or Corporate ID.

**Problem:** You have multiple services running on different ports/containers, but you only want ONE public entry point (DNS name) that intelligently routes traffic based on the URL path.

### 💡 What is ALB?
The Application Load Balancer functions at **Layer 7 (Application Layer)** of the OSI model. It allows for intelligent routing decisions based on the content of the request, such as HTTP headers, cookies, and URL paths.

### 🚀 Why use it? (Benefits)
*   **Smart Routing:** Can route `/api` to one server and `/web` to another (Path-based routing).
*   **Security:** Integrates directly with **AWS WAF** to block attacks and handles SSL/TLS termination.
*   **Microservices Ready:** Perfect for containerized applications (ECS/EKS) where dynamic ports are used.
*   **Auth Support:** Can natively authenticate users (OIDC, SAML, LDAP) before requests reach your app.

### ✅ When to use ALB?
*   [x] Your application is HTTP, HTTPS, or HTTP/2 based.
*   [x] You need routing based on URL path (`/images`) or hostname (`api.app.com`).
*   [x] You are using Docker containers (ECS/EKS) or Lambda functions.
*   [x] You need WebSockets support.

---

## 2. Network Load Balancer (NLB)
**Layer 4 (TCP/UDP)**

### 🎮 The Scenario
Imagine you are building a **real-time multiplayer game** or a **high-frequency trading platform**.
*   The application uses a custom protocol over TCP or UDP (not HTTP).
*   Latency is critical; milliseconds matter.
*   You expect **millions of requests per second** during peak times.
*   Your corporate firewall requires a single, **static IP address** that never changes to whitelist traffic.

**Problem:** ALB is too slow for this (because it inspects headers) and doesn't support static IPs or UDP traffic.

### 💡 What is NLB?
The Network Load Balancer functions at **Layer 4 (Transport Layer)**. It doesn't look at the content of the message; it simply forwards packets based on IP protocol data. This makes it incredibly fast.

### 🚀 Why use it? (Benefits)
*   **Extreme Performance:** Capable of handling millions of requests per second with ultra-low latency.
*   **Static IPs:** Automatically provides a static IP per Availability Zone (ALB does not do this).
*   **Non-HTTP Protocols:** Ideal for databases, gaming servers, email servers (SMTP), or custom protocols.
*   **Source IP Preservation:** The backend server sees the original client's IP address naturally.

### ✅ When to use NLB?
*   [x] You need extreme performance and low latency.
*   [x] Your application uses non-HTTP protocols (TCP, UDP, TLS).
*   [x] You need a Static IP address for whitelisting.
*   [x] You are using a PrivateLink endpoint service.

---

## 3. Gateway Load Balancer (GWLB)
**Layer 3/4 (Gateway + LB)**

### 🛡️ The Scenario
Imagine you are a Security Engineer for a large enterprise.
*   You have 100 different VPCs.
*   You require **ALL** traffic entering or leaving these VPCs to pass through a specific **Firewall Appliance** (like Palo Alto, Cisco, or Fortinet) for deep packet inspection.
*   You need to scale these firewall appliances up and down as traffic changes.

**Problem:** Managing routing tables to force traffic through firewalls in every VPC is complex and hard to scale.

### 💡 What is GWLB?
The Gateway Load Balancer acts as a "bump-in-the-wire." It transparently passes traffic to a fleet of virtual security appliances. It combines a transparent network gateway (to route traffic) and a load balancer (to distribute traffic/health check appliances).

### 🚀 Why use it? (Benefits)
*   **Simplicity:** Makes inserting security appliances into your network topology easy.
*   **High Availability:** Automatically reroutes traffic if a firewall appliance fails.
*   **Scalability:** Allows you to scale your fleet of virtual firewalls horizontally.

### ✅ When to use GWLB?
*   [x] You are deploying third-party virtual appliances (Firewalls, IDS/IPS).
*   [x] You need transparent traffic inspection.
*   [x] You want to centralize security inspection for multiple VPCs.

---

## 4. Classic Load Balancer (CLB)
**Legacy (Layer 4 & 7)**

### 🏛️ The Scenario
You are maintaining a **legacy application** built in 2013 inside an "EC2-Classic" network (which is now retired/retiring). The application relies on older, specific behaviors of the original load balancer.

### 💡 What is CLB?
The original AWS Load Balancer. It operates at both Layer 4 and Layer 7 but lacks modern features like path-based routing, container support, or static IPs.

### ⚠️ Note on Usage
**Do not use CLB for new applications.** It is considered "Previous Generation." Always prefer ALB or NLB.

---

## ⚡ Quick Decision Guide

| Use Case | Recommended LB |
| :--- | :--- |
| **Websites, Microservices, APIs** | **ALB** (Application Load Balancer) |
| **Gaming, Trading, Databases** | **NLB** (Network Load Balancer) |
| **Firewalls, Intrusion Detection** | **GWLB** (Gateway Load Balancer) |
| **Legacy Applications** | **CLB** (Classic Load Balancer) |

---

## 📊 Full Comparison Table

| Feature | Application LB (ALB) | Network LB (NLB) | Gateway LB (GWLB) | Classic LB (CLB) |
| :--- | :--- | :--- | :--- | :--- |
| **OSI Layer** | **Layer 7** (Application) | **Layer 4** (Transport) | **Layer 3/4** (Gateway) | Layer 4/7 |
| **Protocols** | HTTP, HTTPS, gRPC, WebSockets | TCP, UDP, TLS | IP (GENEVE) | HTTP/S, TCP, SSL |
| **Key Routing** | Path (`/img`), Host (`api.`), Headers | IP Address & Port only | Virtual Appliance Targets | Simple Round Robin |
| **Speed** | Fast (Context aware) | **Ultra Fast** (Millions req/sec) | N/A (Transparent) | Moderate |
| **Static IP** | ❌ No (DNS Name only) | ✅ **Yes** (Static IP supported) | ❌ No | ❌ No |
| **Targets** | EC2, Lambda, IP, Containers | EC2, IP, Containers | EC2 Instances (Appliances) | EC2 Instances |
| **Best For** | Microservices, Web Apps, Containers | Real-time Apps, IoT, Gaming | Firewalls, Deep Packet Inspection | Legacy Maintenance |

---

