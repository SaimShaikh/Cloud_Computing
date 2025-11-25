# 📊 AWS CloudWatch Explained


---

## What is CloudWatch?

**AWS CloudWatch** is a monitoring and logging service that helps you **watch your AWS resources** and **see how they're performing**.

Think of CloudWatch as a **doctor's monitoring system** for your cloud infrastructure:
- **Doctor checks vital signs** → CloudWatch checks CPU, memory, disk usage
- **Doctor keeps medical records** → CloudWatch keeps logs of events
- **Doctor sets alerts** → CloudWatch sets alarms to notify you

**Simple Definition:** CloudWatch collects data from your AWS resources, stores it, displays it in charts, and alerts you when something goes wrong.

---

## Why Do We Need CloudWatch?

### Problems Without CloudWatch:

❌ You won't know if your server is down until customers complain
❌ Can't see why your application is slow
❌ No record of what errors happened and when
❌ Can't prove compliance with regulations (GDPR, HIPAA)
❌ Wasting money on resources you don't use
❌ Security breaches go undetected

### Solutions With CloudWatch:

✅ See performance in real-time (before problems happen)
✅ Find performance issues quickly
✅ Complete record of all events (for debugging)
✅ Meet compliance requirements
✅ Optimize spending by seeing actual usage
✅ Detect suspicious activity

---

## When to Use CloudWatch?

| Situation | Use CloudWatch? |
|-----------|-----------------|
| **Production server is slow** | YES - Find the bottleneck |
| **App crashed - need to know why** | YES - Check logs and errors |
| **Need to prove who accessed data** | YES - Track all API calls |
| **Want to auto-scale when CPU is high** | YES - Set alarm to trigger scaling |
| **Compliance audit asking for records** | YES - Generate audit report |
| **Database using too much memory** | YES - Monitor and set alert |
| **Want email when disk is full** | YES - Create alarm with notification |
| **Every production environment** | ALWAYS - Mandatory for reliability |

---

## Benefits of CloudWatch

### 🚀 Performance Optimization
- See exactly which parts of your application are slow
- CPU, memory, disk, network usage in real-time
- Find bottlenecks before users complain
- Make decisions based on real data

**Example:** "Database queries are taking 5 seconds instead of 500ms" → CloudWatch shows this → You optimize queries

---

### 🛡️ Security and Troubleshooting
- Complete record of all errors and events
- Know exactly when problems started
- Track who accessed what resources
- Respond to issues immediately

**Example:** "Website went down at 3:45 PM" → Check CloudWatch logs → See the exact error message

---

### 💰 Cost Optimization
- See which resources use the most
- Find wasteful resources running unused
- Auto-scale based on actual demand
- Prevent unexpected bills

**Example:** "Large server running 24/7 with 5% CPU usage" → CloudWatch shows low usage → Downsize or stop it

---

### 📋 Compliance and Auditing
- Meet GDPR, HIPAA, PCI-DSS requirements
- Prove security controls are working
- Complete audit trails
- Answer "who did what and when" questions

**Example:** Auditor asks "Who accessed customer database?" → Show CloudWatch logs with exact user, time, and IP

---

### ⚡ Proactive Issue Detection
- Set alarms for concerning metrics
- Get notified before problems affect users
- Automatic actions (auto-scaling, remediation)
- Reduce downtime significantly

**Example:** "Alert when CPU > 80%" → Get notified → Add more capacity before performance degrades

---

## CloudWatch Components

CloudWatch has **4 main parts** you need to understand:

### 1. 📈 **Metrics**
**What:** Numbers that measure how your resources are performing

**Examples:**
- CPU Utilization: 65% (How much of the processor is being used)
- Memory Usage: 2048 MB (How much RAM is being used)
- Network In: 1024 bytes (Data coming into your server)
- Request Count: 1,500 requests/minute (How many requests hit your server)

**Where they come from:**
- AWS services automatically send metrics (EC2, RDS, Lambda, etc.)
- Or install CloudWatch Agent on servers to collect custom metrics

**How to view:**
- Graphs and charts in CloudWatch console
- Real-time dashboards
- Export for analysis

---

### 2. 📝 **Logs**
**What:** Text records of everything that happens in your applications and systems

**Examples:**
```
[2025-11-25 12:30:45] INFO - Application started
[2025-11-25 12:30:46] ERROR - Database connection failed
[2025-11-25 12:30:47] INFO - Retrying connection attempt 1
[2025-11-25 12:30:48] INFO - Connection successful
[2025-11-25 13:45:22] WARNING - High memory usage detected: 85%
```

**Where they come from:**
- Application logs (your code prints messages)
- System logs (OS events, errors)
- AWS service logs (API calls, configuration changes)

**How to store:**
- CloudWatch Logs (default, searchable)
- S3 (cheaper for long-term storage)
- Data retention (keep for days, months, or years)

---

### 3. 🚨 **Alarms**
**What:** Automated alerts that notify you when something goes wrong

**How they work:**
1. Monitor a metric continuously
2. When metric crosses a threshold (like CPU > 80%)
3. Alarm triggers and sends you a notification

**Where to send notifications:**
- Email
- SMS (text message)
- Slack
- SNS topic (trigger Lambda, auto-scaling, etc.)

**Example Alarm:**
```
Alarm Name: High CPU Utilization
Metric: CPUUtilization
Threshold: > 80%
Duration: 5 minutes
Action: Send email to admin@company.com
        AND trigger auto-scaling to add more servers
```

---

### 4. 📊 **Dashboards**
**What:** Visual displays showing your most important metrics

**What you can put on a dashboard:**
- Line graphs (CPU over time)
- Bar charts (request counts)
- Gauge widgets (current value)
- Alarm status (green = OK, red = alarm triggered)
- Log insights results

**Why use dashboards:**
- One-page overview of your entire system
- No need to navigate through multiple screens
- Share with team members
- Display on office screens for visibility

**Example Dashboard:**
```
Top: Server CPU, Memory, Disk Status (Gauges)
Middle: Request rate last 24 hours (Line graph)
Bottom: Last 10 errors (Text log display)
Right: Alarm status (3 green = OK, 0 red = alarms triggered)
```

---

## How CloudWatch Works

### The CloudWatch Flow (Simple Version)

```
1. COLLECT
   ↓
   AWS resources (EC2, RDS, Lambda) automatically send metrics
   Applications send logs
   ↓

2. STORE
   ↓
   CloudWatch receives and stores all data
   Metrics kept for 15 months
   Logs kept forever (configurable)
   ↓

3. ANALYZE
   ↓
   Create alarms based on metric thresholds
   Search and query logs
   Create charts and visualizations
   ↓

4. ACTION
   ↓
   Alarms trigger notifications (email, SMS, Slack)
   Auto-scaling triggers
   Lambda functions execute
   Automated responses to problems
```

### Detailed Flow

**Step 1: Data Collection**
```
EC2 Instance → Sends CPU, Memory metrics every 5 minutes (or 1 minute detailed)
RDS Database → Sends query count, connections metrics
Lambda Function → Sends execution count, duration, errors
Application → Logs messages (using CloudWatch agent)
```

**Step 2: Data Storage**
```
CloudWatch ← Receives all metrics and logs
           ← Stores in central repository
           ← Organizes by namespace (EC2, RDS, Lambda, etc.)
           ← Keeps metrics for 15 months
```

**Step 3: Visualization**
```
CloudWatch Console → Display metrics as graphs
                  → Show real-time dashboards
                  → List recent logs
                  → Show alarm status
```

**Step 4: Alerting**
```
CloudWatch monitors metrics constantly
   ↓
When CPU > 80% for 5 minutes
   ↓
Alarm triggers
   ↓
Send email to ops@company.com
Send notification to Slack #alerts channel
Trigger auto-scaling to add 2 more servers
```

---

## Step-by-Step: How to Use CloudWatch

### Scenario: Monitor EC2 Server CPU Usage

#### Step 1: Access CloudWatch Console

```
1. Go to AWS Management Console
2. Search for "CloudWatch"
3. Click on CloudWatch service
4. You're now in CloudWatch dashboard
```

#### Step 2: View Existing Metrics

```
1. In left menu, click "Metrics"
2. Click "EC2"
3. Click "Per-Instance Metrics"
4. Find your EC2 instance (e.g., "prod-web-server-1")
5. Select "CPUUtilization" checkbox
6. You'll see a graph showing CPU usage over time
```

#### Step 3: Create an Alarm

```
1. In left menu, click "Alarms"
2. Click "Create Alarm"
3. Select metric: EC2 → Per-Instance Metrics → CPUUtilization
4. Choose your instance
5. Set threshold:
   - When: CPUUtilization
   - Is: Greater than
   - Threshold: 80
   - For: 5 minutes
6. Configure notification:
   - Send to: Email
   - Email address: admin@company.com
7. Name the alarm: "High CPU Usage - prod-web-server-1"
8. Click "Create Alarm"
```

#### Step 4: Create a Dashboard

```
1. In left menu, click "Dashboards"
2. Click "Create Dashboard"
3. Name: "Production Servers Overview"
4. Add widgets:
   - Click "Add widget"
   - Choose "Line graph"
   - Metric: EC2 → CPUUtilization
   - Select your instances
   - Click "Add to dashboard"
5. Add more widgets for:
   - Memory usage
   - Network traffic
   - Request count
6. Click "Save"
7. Now you have one-page view of all important metrics
```

#### Step 5: Monitor Logs

```
1. In left menu, click "Log Groups"
2. Look for your application log group (e.g., "/aws/lambda/my-function")
3. Click on it to see recent logs
4. View log streams (individual executions/runs)
5. Search for errors using "Filter events":
   - Type "ERROR" to find all errors
   - Type specific text to find issues
```

#### Step 6: Set Up Log Alarms

```
1. In left menu, click "Log Groups"
2. Select your log group
3. Click "Create Metric Filter"
4. Pattern: [ERROR]
5. Filter name: "Application Errors"
6. Click "Create Metric Filter"
7. Click "Create Alarm"
8. When errors > 5 per minute, send email
9. Name: "High Error Rate Alert"
10. Create alarm
```

---

## Real-World Scenarios

### Scenario 1: Website Suddenly Slow

**Situation:** Your e-commerce website becomes very slow (3-5 seconds to load)

**What to do:**

1. **Check CloudWatch Dashboard**
   ```
   Open your dashboard
   Look at CPU, Memory, Disk usage
   If all normal → problem elsewhere
   If CPU 95% → server is overloaded
   ```

2. **Check Metrics**
   ```
   If CPU high → Need to:
     - Optimize code
     - Upgrade instance
     - Add load balancer to distribute traffic
   If Memory high → 
     - Application has memory leak
     - Increase RAM
     - Restart application
   ```

3. **Check Logs**
   ```
   CloudWatch Logs → Search for "ERROR"
   Look at timestamps around when slowness started
   Find specific error messages
   ```

4. **Result**
   ```
   Found: Database query taking 4 seconds
   Fixed: Added database index
   Performance: Back to normal (500ms)
   ```

---

### Scenario 2: Auto-Scaling Setup

**Situation:** Your traffic varies throughout the day
- 10 AM: 100 users → Need 2 servers
- 1 PM: 10,000 users → Need 20 servers
- 8 PM: 100 users → Need 2 servers

**Manual approach (BAD):**
- Hire person to manually add/remove servers
- Can't react fast enough
- Expensive
- Error-prone

**CloudWatch approach (GOOD):**
1. **Create alarm:** "Scale UP when CPU > 70%"
   - Add 5 more servers automatically
   
2. **Create alarm:** "Scale DOWN when CPU < 20%"
   - Remove 5 servers automatically

3. **Result:**
   - Automatically scales based on traffic
   - Always optimal number of servers
   - Saves money during low traffic
   - Handles spikes automatically

---

### Scenario 3: Compliance Audit

**Situation:** Auditor asks: "Who accessed the customer database?"

**Manual approach (BAD):**
- No records
- Can't answer the question
- Fail audit
- Risk losing customers

**CloudWatch approach (GOOD):**
1. **Open CloudTrail** (CloudWatch integration)
2. **Search for database access events**
3. **Results show:**
   ```
   User: john.doe
   Time: 2025-11-25 14:30:45
   Action: RDS:DescribeDBInstances
   IP: 203.0.113.45
   Status: Success
   ```
4. **Generate audit report**
   - Show exactly who, when, what, from where
   - Pass audit
   - Prove compliance

---

### Scenario 4: Cost Optimization

**Situation:** AWS bill increased 40% - why?

**Investigation with CloudWatch:**

1. **Check metrics for each service**
   ```
   EC2 instances: CPU = 5% (very low!)
   RDS database: Connections = 2 (very low!)
   Lambda: Invocations = 100/day (low)
   S3: Data = 500 GB (normal)
   ```

2. **Find the problem**
   ```
   EC2 CPU 5% but running 24/7 on expensive instance type
   Wasting money on unused resources!
   ```

3. **Take action**
   ```
   Downsize instance (smaller = cheaper)
   Or stop instance during non-business hours
   Use auto-scaling to add/remove as needed
   ```

4. **Result**
   ```
   Reduced bill by 35%
   Performance unchanged (still 5% CPU)
   Money saved for other projects
   ```

---

### Scenario 5: Finding Critical Bug

**Situation:** Customers report error "Payment failed" after deployment

**Steps:**

1. **Check CloudWatch Logs**
   ```
   Filter for "Payment failed"
   Find log entries around deployment time (2 PM)
   See error: "Connection timeout to payment gateway"
   ```

2. **Check metrics**
   ```
   At exactly 2 PM, latency jumped from 200ms to 5000ms
   Database connections increased from 10 to 500
   New code must be creating too many connections
   ```

3. **Roll back**
   ```
   Revert deployment to previous version
   Latency drops back to 200ms
   Errors stop
   ```

4. **Fix and redeploy**
   ```
   Fix connection pooling in code
   Redeploy new version
   Monitor metrics to ensure problem solved
   Errors remain at 0
   ```

---

### Scenario 6: Database Performance Issues

**Situation:** RDS database is slow and customers can't access accounts

**Investigation:**

1. **Check CloudWatch metrics for RDS**
   ```
   Database Connections: 1,000 (normal: 50)
   CPU: 95% (normal: 30%)
   I/O Operations: Very high
   Query Execution Time: 10 seconds (normal: 500ms)
   ```

2. **Check logs**
   ```
   CloudWatch Logs show many slow queries
   One SELECT query taking 30 seconds
   Without proper indexes
   ```

3. **Find the culprit**
   ```
   Recently deployed new report feature
   Uses slow query without index
   Thousands of users running report simultaneously
   ```

4. **Quick fix**
   ```
   Disable new feature temporarily
   Connections drop to 50
   CPU back to 30%
   Database performance normal
   ```

5. **Permanent fix**
   ```
   Developer adds database index
   Query now takes 100ms instead of 30 seconds
   Re-enable feature
   Monitor metrics to confirm improvement
   ```

---

## CloudWatch vs Grafana vs Datadog

### Comprehensive Comparison Table

| Feature | **AWS CloudWatch** | **Grafana** | **Datadog** |
|---------|-----------------|-----------|-----------|
| **Primary Purpose** | AWS monitoring & logging | Data visualization | Full-stack monitoring platform |
| **Data Collection** | Built-in for AWS services | Requires external data sources | Built-in agents (700+ integrations) |
| **Ease of Setup** | Very easy (AWS native) | Moderate (need to connect data sources) | Easy (agents included) |
| **Learning Curve** | Low | Moderate | Steeper |
| **Multi-Cloud Support** | AWS only | Yes (with proper config) | Yes (best for multi-cloud) |
| **Dashboards** | Basic dashboards | Highly customizable | Advanced dashboards |
| **Visualization** | Basic charts | Superior customization | Excellent visualizations |
| **Alerting** | Basic threshold alerts | Basic (needs plugins for advanced) | Advanced AI-powered anomaly detection |
| **Log Management** | Built-in CloudWatch Logs | Requires Loki or ELK | Built-in (unified logs + metrics) |
| **APM (App Performance)** | Built-in | Via plugins | Built-in (comprehensive) |
| **Distributed Tracing** | X-Ray integration | Via plugins/integrations | Built-in (excellent) |
| **Cost Model** | Pay-per-metric, pay-per-GB logs | Open-source free / Cloud pricing | Per-host, per-metric (expensive at scale) |
| **Starting Cost** | Very low | Free (open-source) | Moderate to high |
| **Scaling Cost** | Increases with volume | Stays low | Can become very expensive |
| **Customization** | Limited | High | Moderate to high |
| **Query Language** | CloudWatch Insights (limited) | PromQL, Loki with Grafana | Datadog Query Language |
| **Community** | AWS team support | Large open-source community | Enterprise support |
| **Open Source** | No | Yes | No (SaaS only) |
| **On-Premises Option** | No | Yes | No |
| **Data Retention** | 15 months metrics, unlimited logs | Configurable | Configurable |
| **Security Features** | IAM integration | Basic | Advanced (threat detection, etc.) |
| **Incident Management** | Basic | Via integrations | Built-in (comprehensive) |
| **Machine Learning** | Limited | Limited | Advanced anomaly detection |

---

### When to Use Which Tool?

#### **Use CloudWatch If:**
✅ You use **AWS only** (no multi-cloud)
✅ You want **simple, quick setup** with minimal config
✅ You're **cost-conscious** and have low to moderate monitoring needs
✅ You need **tight AWS integration** (IAM, SNS, auto-scaling)
✅ Your team is **small** and doesn't need advanced analytics
✅ You want **built-in AWS service metrics** automatically
✅ **Scenario:** "I have a few EC2 instances and want basic monitoring"

**Cost:** $0.50-2 per 1GB ingested logs + small metric costs

---

#### **Use Grafana If:**
✅ You need **highly customizable dashboards**
✅ You're in a **multi-cloud environment** (AWS, GCP, Azure, on-prem)
✅ You want **flexibility** and don't mind setup complexity
✅ You need **open-source with full control**
✅ You have **developer-heavy teams** that love customization
✅ You want **cost-effective** solution (open-source free)
✅ You're comfortable **managing infrastructure**
✅ **Scenario:** "I use AWS, GCP, and on-premise servers. I need one dashboard for all"

**Cost:** Free (open-source) or $50-200/month (Cloud version) + infrastructure costs

---

#### **Use Datadog If:**
✅ You need **all-in-one observability** (metrics + logs + traces + APM)
✅ You're **multi-cloud and want single vendor**
✅ You need **advanced AI/ML alerting**
✅ You want **minimal setup and maintenance**
✅ You need **incident management** built-in
✅ You can afford **higher costs** for premium features
✅ You need **enterprise support**
✅ **Scenario:** "We use AWS, GCP, and need one platform for everything with advanced analytics"

**Cost:** $15-32+ per host/month (can be $1000s/month at scale)

---

### Quick Decision Flowchart

```
Question: Which monitoring tool should I use?

1. Is your infrastructure AWS-only?
   → YES → Use CloudWatch (cheapest, fastest setup)
   → NO → Go to question 2

2. Do you need multi-cloud with customizable dashboards?
   → YES → Use Grafana (most flexible, open-source)
   → NO → Go to question 3

3. Do you need all-in-one with advanced analytics + incident management?
   → YES → Use Datadog (most features, priciest)
   → NO → CloudWatch is sufficient

4. Do you have budget constraints?
   → YES → Use Grafana (open-source free)
   → NO → Use Datadog (best all-around)
```

---

### Feature Comparison by Use Case

#### **For AWS-Only Deployments**

| Feature | CloudWatch | Grafana | Datadog |
|---------|-----------|---------|---------|
| Auto-discovery of AWS resources | ✅ Best | ⚠️ Manual | ✅ Good |
| EC2/RDS/Lambda metrics | ✅ Built-in | ❌ Need agent | ✅ Good |
| Alarms to auto-scaling | ✅ Native | ❌ No | ✅ Via API |
| Total cost (small team) | ✅ $50-200/mo | ✅ $0 | ❌ $500+/mo |
| **Winner** | **CloudWatch** | Grafana | Datadog |

---

#### **For Multi-Cloud Environments**

| Feature | CloudWatch | Grafana | Datadog |
|---------|-----------|---------|---------|
| AWS monitoring | ✅ Best | ✅ Good | ✅ Good |
| GCP monitoring | ❌ Limited | ✅ Good | ✅ Good |
| Azure monitoring | ❌ Limited | ✅ Good | ✅ Good |
| On-premises | ❌ No | ✅ Best | ✅ Good |
| Single dashboard for all | ❌ No | ✅ Yes | ✅ Yes |
| **Winner** | CloudWatch | **Grafana** | Datadog |

---

#### **For Enterprise With Large Budget**

| Feature | CloudWatch | Grafana | Datadog |
|---------|-----------|---------|---------|
| All-in-one platform | ⚠️ Partial | ❌ Partial | ✅ Yes |
| Advanced APM | ⚠️ Basic | ⚠️ Via plugins | ✅ Best |
| AI/ML alerting | ❌ No | ❌ No | ✅ Yes |
| Incident management | ❌ No | ❌ No | ✅ Yes |
| Enterprise support | ⚠️ Basic | ⚠️ Community | ✅ Yes |
| **Winner** | CloudWatch | Grafana | **Datadog** |

---

### Cost Comparison Example

**Scenario:** Monitor 100 EC2 instances + 10 databases for 1 month

#### CloudWatch Cost:
```
Metrics: 110 instances × 10 metrics/instance × 0.30 = $330
Logs: 100 GB ingested × 0.50 = $50
Total: ~$380/month
```

#### Grafana Cost (Cloud):
```
Monthly subscription: $100
Infrastructure (3 servers): $300
Total: ~$400/month
```

#### Datadog Cost:
```
110 hosts × $25/month = $2,750
Logs: 100 GB × $0.10 = $10
Total: ~$2,760/month
```

**Cheapest:** CloudWatch ($380)
**Flexible:** Grafana ($400)
**Most Features:** Datadog ($2,760)

---

## Quick Reference

### Common Metrics to Monitor

| Metric | What It Means | Good Value | Bad Value |
|--------|--------------|-----------|-----------|
| **CPUUtilization** | CPU usage % | 20-50% | > 80% (bottleneck) or 0% (wasted) |
| **Memory** | RAM usage % | 30-60% | > 85% (too much) or 5% (wasted) |
| **NetworkIn** | Incoming data rate | Depends on app | Abnormally high = attack or issue |
| **NetworkOut** | Outgoing data rate | Depends on app | Abnormally high = issue |
| **DiskUsage** | Disk space used % | < 70% | > 90% (will fill up) |
| **Latency** | Response time in ms | < 500ms | > 2000ms (slow) |
| **RequestCount** | Requests per minute | Normal for your app | Sudden spike = issue |
| **ErrorCount** | Errors per minute | 0-5 | > 50 = something broke |

---

### Common Thresholds for Alarms

| Metric | Alert Threshold |
|--------|-----------------|
| CPU Utilization | > 80% for 5 minutes |
| Memory Usage | > 85% for 5 minutes |
| Disk Usage | > 90% for any duration |
| Error Rate | > 10 errors per minute |
| Latency | > 2 seconds for 5 minutes |
| Request Errors | > 5% of total requests |
| Network Errors | > 0 for 1 minute |
| Database Connections | > 80% of max connections |

---

### Quick Commands (AWS CLI)

```bash
# List all metrics
aws cloudwatch list-metrics

# Get CPU metric for an instance
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --start-time 2025-11-25T00:00:00Z \
  --end-time 2025-11-25T23:59:59Z \
  --period 3600 \
  --statistics Average

# List all alarms
aws cloudwatch describe-alarms

# Create an alarm
aws cloudwatch put-metric-alarm \
  --alarm-name HighCPUUsage \
  --alarm-description "Alert when CPU > 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold
```

---

## Best Practices

### ✅ DO

1. **Enable detailed monitoring for critical resources**
   - 1-minute granularity instead of 5-minute
   - Catch issues faster

2. **Create meaningful alarms**
   - Base thresholds on your actual baseline
   - Not too sensitive (prevent alert fatigue)
   - Not too loose (miss real issues)

3. **Set up multiple notification channels**
   - Email for important alerts
   - Slack for team notifications
   - PagerDuty for on-call integration

4. **Use descriptive names**
   - Alarm: "HighCPU-prod-db-server-1" (good)
   - Not: "Alarm1" (bad)

5. **Create dashboards per team**
   - DevOps team: Infrastructure metrics
   - Application team: Application metrics
   - Database team: Database metrics
   - Finance team: Cost metrics

6. **Set appropriate log retention**
   - Production: 30-90 days minimum
   - Sensitive data: 1+ year
   - Development: 7-14 days (save costs)

7. **Monitor the monitors**
   - Make sure agents are sending data
   - Check if alarms are triggering correctly
   - Regular audits

### ❌ DON'T

1. **Don't ignore alerts**
   - Set alarms to investigate immediately
   - Investigate root cause, not just symptoms

2. **Don't create too many alarms**
   - Causes "alert fatigue"
   - You'll ignore real problems
   - Create alarms for real issues only

3. **Don't set generic thresholds**
   - Every application is different
   - Baseline 80% CPU for 5 minutes for your app
   - Don't use industry defaults

4. **Don't forget cost of logs**
   - High volume logs get expensive
   - Set retention policies
   - Archive old logs to S3

5. **Don't monitor everything**
   - Focus on key metrics
   - Business-critical services
   - Not every single metric

6. **Don't forget security**
   - Limit CloudWatch access via IAM
   - Don't expose logs publicly
   - Encrypt sensitive log data

---

