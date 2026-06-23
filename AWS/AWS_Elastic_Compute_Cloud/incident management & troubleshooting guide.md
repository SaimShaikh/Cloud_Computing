
# Incident Management & Troubleshooting 

## Cloud Engineer / DevOps Engineer / Production Support Engineer

---



# 1. Introduction to Incident Management

## What is an Incident?

An incident is any unplanned interruption or degradation of a service.

Examples:

* Website Down
* API Returning Errors
* Database Connection Failure
* EC2 Instance Unreachable
* High CPU Utilization
* SSL Certificate Expired

---

## Incident Severity Matrix

| Severity | Impact                      | Example              |
| -------- | --------------------------- | -------------------- |
| P1       | Complete Business Outage    | Payment Gateway Down |
| P2       | Major Functionality Impact  | Login Failure        |
| P3       | Partial Service Degradation | Slow Reports         |
| P4       | Minor Issue                 | UI Alignment Issue   |

---

# 2. Incident Lifecycle

```bash

Alert Triggered
↓
Incident Created
↓
Initial Assessment
↓
Investigation
↓
Root Cause Identification
↓
Resolution
↓
Validation
↓
Communication
↓
RCA
↓
Preventive Action

```
---

# 3. Incident Detection & Initial Assessment

## Objectives

* Understand impact
* Determine severity
* Start investigation

## Sources of Detection

### Monitoring

* CloudWatch
* Prometheus
* Grafana
* Datadog
* Site24x7

### User Reports

* Service Desk
* Email
* Slack
* Teams

---

## First Questions to Ask

### What is impacted?

Examples:

* Website
* API
* Database
* Entire Environment

### When did it start?

### Any recent changes?

Examples:

* Deployment
* Infrastructure Change
* Security Group Modification

---

## Real Incident

### Alert

CPU > 95%

### Investigation

CloudWatch shows CPU spike after deployment.

### Root Cause

Infinite loop introduced in code.

### Resolution

Rollback deployment.

---

# 4. Application & Website Validation

## Browser Validation

Verify:

* Website Opens
* Login Works
* API Responds

---

## Curl Validation

curl -I https://example.com

Expected:

HTTP/1.1 200 OK

---

## DNS Validation

nslookup example.com

dig example.com

Verify:

* Correct IP
* Correct DNS Record

---

## SSL Validation

openssl s_client -connect example.com:443

Check:

* Expiration Date
* Certificate Chain
* TLS Version

---

## Real Incident

### Symptom

Website Unreachable

### Root Cause

Expired SSL Certificate

### Resolution

Renew Certificate

---

# 5. HTTP Error Code Analysis

## 2xx Success

200 OK

201 Created

204 No Content

---

## 3xx Redirection

301 Moved Permanently

302 Found

### Real Incident

Redirect Loop

Cause:

Misconfigured NGINX Redirect Rule

---

## 4xx Client Errors

400 Bad Request

401 Unauthorized

403 Forbidden

404 Not Found

429 Too Many Requests

### Real Incident

403 Access Denied

Cause:

Incorrect IAM Policy

---

## 5xx Server Errors

500 Internal Server Error

502 Bad Gateway

503 Service Unavailable

504 Gateway Timeout

### Real Incident

502 Bad Gateway

Architecture:

ALB
↓
NGINX
↓
NodeJS

NodeJS crashed.

ALB returns 502.

---

# 6. AWS Infrastructure Validation

## EC2 Validation

Check:

* Running State
* Status Checks
* System Logs

---

## Security Group Validation

Verify:

* Port 80
* Port 443
* Port 22

---

## Route Table Validation

Verify:

* Internet Gateway
* NAT Gateway

---

## Load Balancer Validation

Verify:

* Healthy Targets
* Listener Rules

---

## Route53 Validation

Verify:

* A Records
* CNAME Records

---

## Real Incident

Website Down

Root Cause:

Security Group Removed Port 443

Resolution:

Restore Rule

---

# 7. EC2 Diagnosis & Recovery

## CPU Troubleshooting

top

htop

---

## Memory Troubleshooting

free -h

vmstat

---

## Disk Troubleshooting

df -h

du -sh *

---

## Process Troubleshooting

ps -ef

---

## Real Incident

CPU 100%

Cause:

Java Process Memory Leak

Resolution:

Restart Service

Fix Code

---

# 8. OS-Level Troubleshooting

## Service Validation

systemctl status nginx

systemctl status docker

---

## Log Validation

journalctl -xe

tail -f /var/log/messages

---

## Network Validation

ss -tulpn

netstat -tulpn

---

## Real Incident

NGINX Not Starting

Cause:

Syntax Error

Command:

nginx -t

Resolution:

Fix Configuration

---

# 16. Incident Resolution & Communication

## During Incident

Example Update

Investigation in progress.

Issue identified.

Next update in 15 minutes.

---

## Resolution Message

Incident Resolved

Service Restored.

Root Cause Identified.

Monitoring Stability.

---

# 17. Root Cause Analysis (RCA)

## RCA Template

### Incident

Website Down

### Root Cause

Memory Leak

### Resolution

Restarted Service

### Preventive Action

Implemented Memory Monitoring

---

# 19. Real Production Incidents

## Incident #1

High CPU

Cause:

Infinite Loop

Impact:

Website Slow

Resolution:

Rollback Deployment

---

## Incident #2

503 Service Unavailable

Cause:

No Healthy Targets

Resolution:

Restart Application

---

## Incident #3

EC2 Unreachable

Cause:

Disk Full

Resolution:

Cleanup Logs

Expand Volume

---

## Incident #4

Database Connection Failure

Cause:

Security Group Misconfiguration

Resolution:

Restore Database Port Access

---


