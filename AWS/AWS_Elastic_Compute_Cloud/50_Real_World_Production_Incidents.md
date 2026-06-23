# 50 Real-World Production Incidents

## Cloud Engineer / DevOps Engineer / Production Support Engineer

---

# Purpose

This document contains common real-world incidents frequently encountered in AWS, Linux, DevOps, and Production Support environments.

Each incident includes:

* Symptoms
* Investigation
* Root Cause
* Resolution
* Prevention

---

# Incident #1 – Website Completely Down

## Symptoms

* Website inaccessible
* Users report outage
* Monitoring alerts triggered

## Investigation

```bash
curl -I https://app.company.com
systemctl status nginx
```

## Root Cause

NGINX service stopped.

## Resolution

```bash
sudo systemctl restart nginx
```

## Prevention

* Configure monitoring
* Enable service auto-start

---

# Incident #2 – Website Very Slow

## Symptoms

* High response times
* User complaints

## Investigation

```bash
top
htop
```

## Root Cause

CPU utilization reached 100%.

## Resolution

* Identify process
* Scale instance
* Optimize application

## Prevention

CloudWatch CPU alarms.

---

# Incident #3 – Login Failure

## Symptoms

```text
401 Unauthorized
```

## Root Cause

Expired JWT token.

## Resolution

Refresh authentication token.

---

# Incident #4 – Website Returning 404

## Root Cause

Application deployment removed static content.

## Resolution

Redeploy application.

---

# Incident #5 – Redirect Loop

## Symptoms

```text
ERR_TOO_MANY_REDIRECTS
```

## Root Cause

Incorrect NGINX redirect configuration.

## Resolution

Correct redirect rules.

---

# Incident #6 – ALB Returns 502 Bad Gateway

## Architecture

```text
ALB
 ↓
NGINX
 ↓
NodeJS
```

## Root Cause

Backend application crashed.

## Resolution

Restart backend service.

---

# Incident #7 – ALB Returns 503 Service Unavailable

## Root Cause

No healthy targets available.

## Resolution

Validate target group health.

---

# Incident #8 – ALB Returns 504 Gateway Timeout

## Root Cause

Database query timeout.

## Resolution

Optimize query performance.

---

# Incident #9 – Target Group Unhealthy

## Root Cause

Incorrect health check path.

## Resolution

Update health check configuration.

---

# Incident #10 – SSL Not Working Behind ALB

## Root Cause

Missing ACM certificate.

## Resolution

Attach certificate to listener.

---

# Incident #11 – EC2 Instance Unreachable

## Investigation

```bash
ping
ssh
```

## Root Cause

Security Group blocked port 22.

## Resolution

Update Security Group.

---

# Incident #12 – EC2 Status Check Failure

## Root Cause

Kernel panic.

## Resolution

Review system logs.

---

# Incident #13 – CPU Utilization 100%

## Root Cause

Infinite loop in application.

## Resolution

Kill process and deploy fix.

---

# Incident #14 – Memory Utilization 100%

## Root Cause

Memory leak.

## Resolution

Restart application and investigate heap usage.

---

# Incident #15 – OOM Killer Triggered

## Investigation

```bash
dmesg | grep kill
```

## Root Cause

Insufficient memory.

## Resolution

Increase memory or optimize application.

---

# Incident #16 – Disk Full

## Investigation

```bash
df -h
```

## Root Cause

Log accumulation.

## Resolution

Clean old logs.

---

# Incident #17 – EBS Volume Full

## Root Cause

Application generated excessive logs.

## Resolution

Increase EBS volume size.

---

# Incident #18 – SSH Connection Timeout

## Root Cause

NACL blocking SSH traffic.

## Resolution

Update NACL rules.

---

# Incident #19 – Random EC2 Reboots

## Root Cause

Underlying host maintenance.

## Resolution

Review AWS Health Dashboard.

---

# Incident #20 – Application Port Not Listening

## Investigation

```bash
ss -tulpn
```

## Root Cause

Application crashed.

## Resolution

Restart application.

---

# Incident #21 – Service Fails to Start

## Investigation

```bash
systemctl status service-name
```

## Root Cause

Configuration error.

## Resolution

Fix configuration.

---

# Incident #22 – Port Already in Use

## Investigation

```bash
lsof -i :8080
```

## Resolution

Stop conflicting process.

---

# Incident #23 – High Load Average

## Investigation

```bash
uptime
```

## Root Cause

CPU saturation.

## Resolution

Optimize workload.

---

# Incident #24 – Large Log File Growth

## Root Cause

Debug logging enabled.

## Resolution

Rotate logs.

---

# Incident #25 – Cron Job Failure

## Investigation

```bash
grep CRON /var/log/syslog
```

## Resolution

Correct cron configuration.

---

# Incident #26 – Domain Not Resolving

## Investigation

```bash
nslookup company.com
```

## Root Cause

Missing DNS record.

---

# Incident #27 – Incorrect DNS Resolution

## Root Cause

Wrong Route53 record.

---

# Incident #28 – DNS Propagation Delay

## Root Cause

High TTL value.

---

# Incident #29 – Route53 Private Zone Failure

## Root Cause

VPC association issue.

---

# Incident #30 – Split Horizon DNS Issue

## Root Cause

Conflicting public and private records.

---

# Incident #31 – SSL Certificate Expired

## Symptoms

```text
Not Secure
```

## Resolution

Renew certificate.

---

# Incident #32 – Certificate Mismatch

## Root Cause

Incorrect SAN entry.

---

# Incident #33 – ACM Auto Renewal Failure

## Root Cause

DNS validation record deleted.

---

# Incident #34 – TLS Handshake Failure

## Root Cause

Unsupported cipher suite.

---

# Incident #35 – HTTPS Redirect Failure

## Root Cause

ALB listener misconfiguration.

---

# Incident #36 – Internet Connectivity Failure

## Root Cause

Missing Internet Gateway route.

---

# Incident #37 – Private EC2 Cannot Access Internet

## Root Cause

Missing NAT Gateway.

---

# Incident #38 – VPC Peering Failure

## Root Cause

Missing route table entries.

---

# Incident #39 – Security Group Misconfiguration

## Root Cause

Required port blocked.

---

# Incident #40 – NACL Blocking Traffic

## Root Cause

Incorrect deny rule.

---

# Incident #41 – EBS Snapshot Failure

## Root Cause

KMS permission issue.

---

# Incident #42 – Snapshot Restore Failure

## Root Cause

Deleted KMS key.

---

# Incident #43 – S3 Access Denied

## Root Cause

IAM policy issue.

---

# Incident #44 – S3 Bucket Not Accessible

## Root Cause

Bucket policy misconfiguration.

---

# Incident #45 – Lifecycle Policy Deleted Data

## Root Cause

Incorrect lifecycle configuration.

---

# Incident #46 – CloudWatch Alarm Not Triggering

## Root Cause

Wrong metric namespace.

---

# Incident #47 – Alert Storm

## Root Cause

Threshold configuration too low.

---

# Incident #48 – Missing CloudWatch Logs

## Root Cause

CloudWatch agent stopped.

---

# Incident #49 – Grafana Dashboard Empty

## Root Cause

Prometheus target unreachable.

---

# Incident #50 – No Alerts Received

## Root Cause

SNS subscription not confirmed.

---

