# 🚀 Centralized Monitoring & Alerting (AWS)

## 📌 Project Overview

This project demonstrates how to build a **centralized monitoring and alerting system** on AWS using:

* Amazon EC2 (Ubuntu)
* Amazon CloudWatch
* CloudWatch Agent
* Amazon SNS

The system collects system metrics and logs, monitors them in real-time, and triggers alerts when thresholds are breached.

---
## 🏗️ Architecture

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/a59b491b-f22c-47f5-8e70-f809100867dc" />





---

## ⚙️ Tech Stack

* AWS EC2
* Amazon CloudWatch
* CloudWatch Agent
* Amazon SNS
* Linux (Ubuntu)

---

## 🔧 Setup Guide

### 1️⃣ Launch EC2 Instance

* OS: Ubuntu
* Instance Type: t2.micro
* Region: ap-south-1

Attach IAM Role:

```
CloudWatchAgentServerPolicy
```

---

### 2️⃣ Install CloudWatch Agent

```bash
sudo apt update -y

wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb

sudo dpkg -i amazon-cloudwatch-agent.deb
```

---

### 3️⃣ Configure CloudWatch Agent

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

#### Select:

* OS → Linux
* Environment → EC2
* Metrics → CPU, Memory, Disk, Network
* Logs:

```
/var/log/syslog
/var/log/auth.log
```

Log Group:

```
/ec2/ubuntu-syslog
```
- Answer all question or add this in sudo vim /opt/aws/amazon-cloudwatch-agent/bin/config.json

```bash
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "cwagent"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/syslog",
            "log_group_name": "/ec2/ubuntu-syslog",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 30
          },
          {
            "file_path": "/var/log/auth.log",
            "log_group_name": "/ec2/ubuntu-authlog",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 30
          }
        ]
      }
    }
  },
  "metrics": {
    "append_dimensions": {
      "AutoScalingGroupName": "${aws:AutoScalingGroupName}",
      "ImageId": "${aws:ImageId}",
      "InstanceId": "${aws:InstanceId}",
      "InstanceType": "${aws:InstanceType}"
    },
    "metrics_collected": {
      "cpu": {
        "measurement": [
          "cpu_usage_idle",
          "cpu_usage_iowait",
          "cpu_usage_user",
          "cpu_usage_system"
        ],
        "metrics_collection_interval": 60,
        "resources": ["*"],
        "totalcpu": false
      },
      "disk": {
        "measurement": [
          "used_percent",
          "inodes_free"
        ],
        "metrics_collection_interval": 60,
        "resources": ["*"]
      },
      "diskio": {
        "measurement": ["io_time"],
        "metrics_collection_interval": 60,
        "resources": ["*"]
      },
      "mem": {
        "measurement": ["mem_used_percent"],
        "metrics_collection_interval": 60
      },
      "swap": {
        "measurement": ["swap_used_percent"],
        "metrics_collection_interval": 60
      }
    }
  }
}
```

---

### 4️⃣ Start CloudWatch Agent

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a fetch-config \
-m ec2 \
-c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json \
-s
```

---

### 5️⃣ Verify Metrics & Logs

Go to AWS Console:

* CloudWatch → Metrics → CWAgent
* CloudWatch → Logs → Log Groups

---

### 6️⃣ Setup SNS (Alert Notification)

* Create Topic: `devops-alerts`
* Type: Standard

Create Subscription:

* Protocol: Email
* Endpoint: Your Email

📩 Confirm subscription via email.

---

### 7️⃣ Create CloudWatch Alarm

* Metric: CPUUtilization
* Threshold: > 70%
* Period: 1 minute
* Evaluation: 2 datapoints

Action:

```
Send notification to SNS (devops-alerts)
```

---

### 8️⃣ Create Dashboard

Add widgets:

* CPUUtilization
* mem_used_percent
* disk_used_percent
* NetworkIn / NetworkOut

---

## 🧪 Testing

Install stress tool:

```bash
sudo apt install stress -y
```

Generate load:

```bash
stress --cpu 2 --timeout 300
```

---

## ✅ Expected Outcome

* CPU usage increases
* CloudWatch alarm triggers
* SNS sends email notification

---

## 📊 Features

* Real-time monitoring
* Log collection
* Automated alerting
* Centralized dashboard

---

## 💬 Interview Explanation

"I implemented centralized monitoring using CloudWatch Agent and configured SNS-based alerting to proactively detect system issues."

---

## ⚡ Key Learnings

* Monitoring vs Alerting
* CloudWatch metrics & logs
* SNS integration
* Incident simulation using stress

---

## 🔥 Future Enhancements

* Terraform automation
* Grafana dashboard
* Log-based alerting
* Multi-instance monitoring

---

## 👨‍💻 Author

Saime Shaikh
DevOps Enthusiast 🚀
