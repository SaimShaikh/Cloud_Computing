
# AWS Route 53 
## Scenario
Imagine you built a website and hosted it on AWS.
Users type www.example.com in the browser, but computers do not understand names, they understand IP addresses.
So the question is: Who converts the name into an IP?
That job is done by AWS Route 53.

---

## What is AWS Route 53?
AWS Route 53 is a DNS (Domain Name System) service provided by AWS.
It converts domain names like example.com into IP addresses so that users can reach your application.
It is highly available, fast, and works globally.

In simple words:
Route 53 is the internet phonebook for your AWS applications.

---

## Why the name Route 53?
- Route means routing traffic
- 53 is the DNS port number

---

## Benefits of AWS Route 53
- Very high availability
- Fast DNS resolution
- Supports health checks and automatic failover
- Easy integration with EC2, Load Balancer, and S3
- Scales automatically
- Pay only for what you use

---

## How DNS works with Route 53 (Flow Diagram)

User Browser
   |
   v
Domain Name (example.com)
   |
   v
AWS Route 53
   |
   v
IP Address (EC2 or Load Balancer)
   |
   v
Application Response

---

## Types of Routing Policies in Route 53

### 1. Simple Routing
Used when you have one resource.
Route 53 always returns the same IP address.

---

### 2. Weighted Routing
Used to split traffic between resources.
You can control traffic percentage.

Example:
Server A → 70%
Server B → 30%

---

### 3. Latency-Based Routing
Routes users to the nearest region.
Helps reduce application delay.

---

### 4. Failover Routing
Used for backup and disaster recovery.
Traffic moves to secondary resource if primary fails.

---

### 5. Geolocation Routing
Routes users based on country or region.

---

### 6. Multi-Value Answer Routing
Returns multiple healthy IP addresses.
Works like basic load balancing.

---

