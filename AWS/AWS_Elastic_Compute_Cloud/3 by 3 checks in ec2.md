# Amazon EC2 Status Checks (3/3 Checks Passed)

## Objective

Understand what **3/3 Checks Passed** means for an Amazon EC2 instance, what each health check verifies, common failure scenarios, and troubleshooting methods.

---

# What is 3/3 Checks Passed?

When an EC2 instance displays:

```
Status Checks
3/3 Checks Passed
```

it means all three health checks have successfully passed, indicating that the EC2 instance and its attached storage are healthy.

The three health checks are:

1. System Status Check
2. Instance Status Check
3. Attached EBS Status Check

---

# Architecture

```
                    +---------------------------+
                    |      AWS Data Center      |
                    +------------+--------------+
                                 |
                     System Status Check
                                 |
                                 v
                    +---------------------------+
                    |       EC2 Instance        |
                    |   (Linux / Windows OS)    |
                    +------------+--------------+
                                 |
                    Instance Status Check
                                 |
                                 v
                    +---------------------------+
                    |     Attached EBS Volume   |
                    +------------+--------------+
                                 |
                     Attached EBS Check
                                 |
                                 v
                     Status Checks = 3/3 Passed
```

---

# 1. System Status Check

## Purpose

Verifies the health of the AWS infrastructure hosting your EC2 instance.

AWS checks:

- Physical host
- Power supply
- Network connectivity
- Hardware components
- Hypervisor
- Storage connectivity

This check is completely managed by AWS.

---

## What it does NOT check

- Linux operating system
- SSH service
- Installed applications
- CPU usage
- Memory usage

---

## Common Failure Causes

- Hardware failure
- Host machine crash
- Network outage
- Power issue
- Hypervisor issue
- AWS infrastructure maintenance

---

## Example

```
System Status Check : Failed
```

Even if your Linux server is healthy, you cannot access it because the physical AWS host has an issue.

---

## Troubleshooting

- Wait for AWS to recover the host.
- Stop and Start the instance (moves the instance to a new physical host for EBS-backed instances).
- Check the AWS Health Dashboard.
- Contact AWS Support if the issue persists.

---

# 2. Instance Status Check

## Purpose

Verifies that the operating system inside the EC2 instance is healthy and responding.

AWS checks:

- Operating system booted successfully
- Kernel is responding
- Network stack is functioning
- Instance is reachable internally

---

## Common Failure Causes

- Kernel panic
- Incorrect `/etc/fstab`
- Corrupted filesystem
- Full disk
- High CPU usage
- Out of memory (OOM)
- Startup script failures
- Broken system configuration

---

## Example

```
Instance Status Check : Failed
```

AWS hardware is healthy, but the operating system has become unresponsive.

---

## Troubleshooting

### Method 1

Use EC2 Serial Console (if enabled).

---

### Method 2

Use AWS Systems Manager (SSM) Session Manager.

---

### Method 3

Detach the root EBS volume:

1. Stop the instance.
2. Detach the root volume.
3. Attach it to another EC2 instance.
4. Repair the filesystem or configuration.
5. Reattach the volume.
6. Start the instance.

---

### Method 4

Restore from an EBS Snapshot if the root volume is corrupted.

---

# 3. Attached EBS Status Check

## Purpose

Checks the health of all attached Amazon EBS volumes.

AWS verifies:

- Storage availability
- Read/Write capability
- Storage backend health
- I/O connectivity

---

## Common Failure Causes

- EBS hardware issue
- Backend storage failure
- Volume unavailable
- Storage connectivity problem

---

## Example

```
Attached EBS Status : Failed
```

The EC2 instance is healthy, but the storage volume has an issue.

---

## Troubleshooting

- Check the EBS Volume State.
- Review Amazon CloudWatch metrics.
- Review EC2 Events.
- Restore the volume from an EBS Snapshot if necessary.
- Replace the affected EBS volume.

---

# Health Check Summary

| Health Check | Checks | Managed By |
|--------------|--------|------------|
| System Status Check | Physical hardware, network, hypervisor | AWS |
| Instance Status Check | Operating system, kernel, internal networking | Customer |
| Attached EBS Status Check | EBS storage health | AWS |

---

# Overall Status

| Result | Meaning |
|---------|---------|
| 3/3 Checks Passed | Everything is healthy |
| 2/3 Checks Passed | One health check failed |
| 1/3 Checks Passed | Two health checks failed |
| 0/3 Checks Passed | Instance is unhealthy |

---

# Where to View Status Checks

## AWS Management Console

```
AWS Console
    ↓
EC2
    ↓
Instances
    ↓
Select an Instance
    ↓
Status Checks
```

You will see:

```
System Status        Passed

Instance Status      Passed

Attached EBS         Passed

Overall Status

3/3 Checks Passed
```

---

# CloudWatch Monitoring

You can also monitor EC2 health using:

- EC2 Status Checks
- CloudWatch Metrics
- CloudWatch Alarms
- EventBridge
- AWS Health Dashboard

---

# Difference Between 2/2 and 3/3 Checks

## Older EC2 Console

```
2/2 Checks Passed

✓ System Status
✓ Instance Status
```

Only two health checks were displayed.

---

## Newer EC2 Console

```
3/3 Checks Passed

✓ System Status
✓ Instance Status
✓ Attached EBS Status
```

AWS now also reports the health of attached EBS volumes, providing better visibility into storage health.

---

