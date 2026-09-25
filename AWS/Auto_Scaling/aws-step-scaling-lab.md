# AWS Step Scaling — End-to-End Hands-On Lab (Verified)

Verified against AWS's official EC2 Auto Scaling documentation. Corrected in three passes: a sequencing error and two missing details from the first draft, a testing-procedure bug found on review, and a production-hardening pass on the launch template and user-data script. Full list at the end.

## What You're Building

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/6ee32240-bec2-4fd9-b21e-493438cda63f" />


**Scale OUT steps** (relative to a single alarm breached at CPU >= 40%):
| CPU range | Action |
|---|---|
| 40–60% | +1 instance |
| 60–80% | +2 instances |
| >80% | +3 instances |

**Scale IN steps** (relative to a single alarm breached at CPU <= 30%):
| CPU range | Action |
|---|---|
| 20–30% | -1 instance |
| <20% | -2 instances |

**Why step scaling and not target tracking:** target tracking just holds CPU near a set point automatically. Step scaling lets you react with different intensity at different severity levels — useful to demonstrate in an interview because it shows you understand *adjustment magnitude*, not just threshold crossing.

---

## Part 1 — VPC

**What:** VPC → Your VPCs → Create VPC → "VPC and more."
- Name: `step-scaling-lab`
- IPv4 CIDR: `10.20.0.0/16`
- AZs: 2, Public subnets: 2, Private subnets: 0, NAT gateways: None

**Why:** ASG instances need internet access (via IGW, not NAT — no private subnets needed) to install nginx and stress tools. Two AZs are required to actually demonstrate the multi-AZ placement the ASG does.

**Verify:** Route table for the public subnets has `0.0.0.0/0 → Internet Gateway`.

---

## Part 2 — Security Groups

**What:**
- `step-scaling-alb-sg`: inbound HTTP 80 from `0.0.0.0/0`, outbound all.
- `step-scaling-ec2-sg`: inbound HTTP 80 from `step-scaling-alb-sg` only, outbound all (needed for apt, SSM API calls, and the IMDSv2 metadata call). **No SSH inbound rule at all.**

**Why:** EC2 should only ever receive HTTP traffic that's already passed through the ALB. Shell access comes from Session Manager (Part 3's `AmazonSSMManagedInstanceCore`), which rides over the AWS API rather than an open inbound port — there's no port 22 rule to leave scoped-to-your-IP-and-forgotten-about later. This is the actual production pattern; a "temporarily open to My IP" SSH rule is exactly the kind of thing that outlives the lab it was created for.

---

## Part 3 — IAM Role for EC2

**What:** IAM → Roles → Create role → AWS service → EC2.
Attach: `CloudWatchAgentServerPolicy`, `AmazonSSMManagedInstanceCore`.
Name: `StepScalingEC2Role`.

**Why:** SSM lets you access the instance via Session Manager without opening SSH at all (optional but cleaner). CloudWatch agent policy is only needed if you go beyond the default EC2 metrics later (CPU utilization is available by default without the agent — you do **not** strictly need this policy just for basic CPU-based step scaling, but it's harmless to attach if you plan to extend the lab with custom metrics like memory).

---

## Part 4 — Launch Template

**What:** EC2 → Launch Templates → Create.
- Name: `step-scaling-launch-template`
- AMI: Ubuntu Server 24.04 LTS
- Instance type: `t3.micro`
- Key pair: none needed — access is via Session Manager only
- Do **not** pin a subnet — let the ASG pick per-AZ
- Security group: `step-scaling-ec2-sg`
- Storage: 8–10 GB gp3
- IAM instance profile: `StepScalingEC2Role`
- **Advanced details → Metadata version: Required (IMDSv2 only)**, Metadata token response hop limit: **1**

**Why enforce IMDSv2 at the launch template level, not just in the script:** setting `HttpTokens: required` on the launch template means the instance itself rejects any IMDSv1-style unauthenticated metadata call — not just "the script happens to use a token." This matters because anything else that lands on the box later (a sidecar, a misconfigured tool, a compromised process) is also blocked from IMDSv1-style SSRF-style metadata theft, which is the actual production reason this setting exists — it's not primarily about your curl call working.

User data:
```bash
#!/bin/bash
set -euo pipefail
exec > >(tee /var/log/user-data.log) 2>&1

echo "== apt update =="
for i in 1 2 3; do apt-get update -y && break || { echo "apt-get update failed, retry $i"; sleep 5; }; done

echo "== installing packages =="
DEBIAN_FRONTEND=noninteractive apt-get install -y nginx stress-ng

echo "== fetching instance metadata (IMDSv2) =="
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
PRIVATE_IP=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/local-ipv4)

if [ -z "$INSTANCE_ID" ]; then
  echo "FATAL: could not retrieve instance metadata, aborting"
  exit 1
fi

echo "== writing public health page (no instance metadata exposed) =="
cat > /var/www/html/index.html <<EOF
<!DOCTYPE html>
<html><head><title>AWS Step Scaling Lab</title></head>
<body>
<h1>AWS Step Scaling Lab</h1>
<p>Application is running.</p>
</body></html>
EOF

echo "== writing internal debug page (for your own testing only, not linked) =="
mkdir -p /var/www/html/debug
cat > /var/www/html/debug/index.html <<EOF
<!DOCTYPE html>
<html><body>
<p>Instance ID: $INSTANCE_ID</p>
<p>Private IP: $PRIVATE_IP</p>
</body></html>
EOF

echo "== enabling and starting nginx =="
systemctl enable nginx
systemctl restart nginx

sleep 2
if ! systemctl is-active --quiet nginx; then
  echo "FATAL: nginx failed to start"
  exit 1
fi

echo "== user-data complete =="
```

**Why each change from the original:**
- `set -euo pipefail` + explicit `exit 1` on failure: a script that silently continues after a failed `apt-get install` gives you an instance that reports healthy (booted fine) but never actually serves traffic — the ASG will happily count it as launched while the target group marks it unhealthy forever, and you'll waste time debugging the wrong layer.
- Logging to `/var/log/user-data.log`: without this, your only way to debug a failed launch is guessing. With it, SSM Session Manager → `cat /var/log/user-data.log` tells you exactly which line failed.
- Retry loop on `apt-get update`: transient mirror/network blips are common enough on fresh boots that a single-shot `apt-get update` is a known flaky point in real launch templates.
- IMDSv2 token flow: matches the `HttpTokens: required` setting above — without this, the script itself would fail once IMDSv2-only is enforced.
- **Instance ID and private IP moved off the public root page to `/debug/`:** the public page an ALB-facing target group serves should not leak internal AWS identifiers. Keep `/debug/` for your own manual checks during the lab (curl it directly from inside the VPC, or over SSM port forwarding) — don't put it behind the public ALB listener if you want this to actually resemble production. If you do want it reachable through the ALB for convenience during the lab, that's a deliberate lab-only tradeoff, not a production pattern — say so if you're demoing this to anyone.

---

## Part 5 — Target Group

**What:** EC2 → Target Groups → Create → target type **Instances**, protocol HTTP:80, VPC `step-scaling-lab`, health check path `/`.

**Why:** Don't manually register instances — the ASG does that automatically once attached. If you register manually, ASG's own attach/detach lifecycle will conflict with it.

---

## Part 6 — Application Load Balancer

**What:** EC2 → Load Balancers → Create → Application Load Balancer.
- Internet-facing, both public subnets, SG `step-scaling-alb-sg`
- Listener HTTP:80 → forward to `step-scaling-tg`

---

## Part 7 — Auto Scaling Group

**What:** EC2 → Auto Scaling Groups → Create.
- Launch template: `step-scaling-launch-template`
- VPC + both public subnets (one per AZ)
- Attach to existing load balancer → `step-scaling-tg`
- Health checks: enable **ELB** health checks (not just EC2 status checks — ELB checks actually verify nginx is serving, EC2 checks only verify the instance is running)
- Health check grace period: 120s
- Group size: Min 2 / Desired 2 / Max 6
- Scaling policy at creation: **choose "No scaling policy"** — you're adding step scaling manually next

**Verify before continuing:** Instance management tab shows 2 instances as `InService`, and Target Group shows 2 healthy targets. If either is red, stop and fix it before touching scaling policies — a scaling test on top of a broken health check just produces noise.

---

## Part 8 — Scale-Out: CloudWatch Alarm First

This is the step the original draft got out of order. **The alarm must exist before the scaling policy can reference it** — you can't create both inside one ASG wizard screen.

**What:** CloudWatch console → Alarms → All alarms → Create alarm.
- Select metric → EC2 → By Auto Scaling Group → find `step-scaling-asg` → `CPUUtilization`
- Statistic: Average, Period: 1 minute
- Condition: Greater/Equal → 40
- Evaluation period: 1 out of 1
- Name: `step-scaling-alarm-high`
- Skip the notification (SNS) step for the lab, or add one if you want an email on trigger.

**Why 1-minute period / 1 evaluation:** production alarms usually use longer periods to avoid flapping. For a lab where you want to *see* the reaction quickly, 1 minute keeps the feedback loop short — just know this is intentionally aggressive, not a production setting.

---

## Part 9 — Scale-Out: Step Scaling Policy

**What:** EC2 → Auto Scaling Groups → `step-scaling-asg` → Automatic scaling tab → Create dynamic scaling policy.
- Policy type: Step scaling
- Policy name: `step-scale-out`
- CloudWatch alarm: select `step-scaling-alarm-high` (the one you just made)
- Add step adjustments:
  - 40–60% → +1
  - 60–80% → +2
  - >80% → +3
- Instance warmup: set to something realistic for your boot + nginx startup time, e.g. 90 seconds

**Why "instance warmup" matters:** while a newly launched instance is in its warmup period, its (near-zero) CPU isn't yet counted toward the group's aggregate metric. Skip this setting and a burst of new instances booting up can drag the average CPU down and trigger a premature scale-in — confusing behavior if you don't know this field exists.

**Note on the console vs. CLI discrepancy:** the console lets you type these as absolute CPU percentages, but AWS stores them internally as offsets relative to the 40% breach threshold (0–20 → +1, 20–40 → +2, 40+ → +3). If you later replicate this with `aws autoscaling put-scaling-policy` or Terraform, you must supply the *relative* bounds, not the absolute ones — the console does that translation for you silently.

---

## Part 10 — Scale-In: CloudWatch Alarm, Then Policy

**What (alarm):** CloudWatch → Create alarm → same metric, Statistic Average, Period 1 min.
- Condition: Less/Equal → 30
- Name: `step-scaling-alarm-low`

**What (policy):** ASG → Automatic scaling → Create dynamic scaling policy.
- Policy type: Step scaling, name `step-scale-in`
- CloudWatch alarm: `step-scaling-alarm-low`
- Steps: 20–30% → -1, <20% → -2

**Verify before testing scale-in:** if desired capacity is already sitting at your minimum (2), a scale-in alarm firing won't do anything — AWS respects the floor. Don't mistake "no scale-in happened" for "the alarm is broken" if you're already at minimum.

---

## Part 11 — Test Scale-Out

1. Connect via Session Manager: EC2 → Instances → select instance → Connect → Session Manager tab → Connect. (No SSH key or open port needed — this is why Part 2 has no port 22 rule.)
2. `nproc` — confirm 2 vCPUs on `t3.micro`.
3. Run: `stress-ng --cpu 2 --cpu-load 70 --timeout 10m`
4. Watch CloudWatch → Metrics → EC2 → By Auto Scaling Group → `CPUUtilization` climb.
5. Watch ASG → Instance management: desired capacity should move 2 → 3 once the alarm transitions to `ALARM` state (allow 1–2 minutes for evaluation + a bit more for the new instance to boot, warm up, and pass health checks).
6. To test the higher step, first stop the running process — `pkill stress-ng` — then start a fresh one at higher load: `stress-ng --cpu 2 --cpu-load 90 --timeout 10m`. Skipping the kill means the first 10-minute job is still running when the second starts, so you get two overlapping stress-ng workloads on 2 vCPUs instead of a clean 90% reading — the CPU number you see won't correspond to the step you think you're testing. Watch capacity move further toward 6 (capped by your max).

**Expected delay chain:** CPU load → metric published (~1 min) → alarm evaluation → alarm state change → scaling policy fires → ASG launches instance → EC2 boot → warmup period → health check passes → `InService`. Total: often 3–5 minutes end to end, not instant.

---

## Part 12 — Test Auto Healing (Separate From Scaling)

**What:** With desired capacity at 2 and both healthy, manually terminate **one** instance (EC2 → Instances → Terminate — not both).

**Why:** this proves the ASG maintains desired capacity independent of the CPU-based policies — a replacement launches even with zero CPU load anywhere. This is the distinction interviewers actually probe: Auto Scaling Group = "how many should exist," Step Scaling = "how much should that number change when a metric crosses a threshold." Conflating the two is the most common mistake people make explaining this system.

---

## Part 13 — Test Scale-In

1. `pkill stress-ng` or let the 10-minute timeout expire.
2. Watch CPU drop in CloudWatch.
3. Once average CPU is ≤30% and the low alarm fires, watch desired capacity step down (respecting your minimum of 2).
4. Check ASG → Instance management → Activity tab for `Terminating an EC2 instance` events.

---

## Part 14 — What to Record While Testing

| Time | CPU | Desired | Running | Action |
|---|---|---|---|---|
| 10:00 | 8% | 2 | 2 | Normal |
| 10:05 | 45% | 3 | 3 | +1 |
| 10:10 | 65% | 5 | 5 | +2 |
| 10:15 | 85% | 6 | 6 | +3, capped at max |
| 10:25 | 15% | 4 | 4 | -2 |
| 10:30 | 10% | 2 | 2 | -2 |

These numbers are illustrative — your actual timings will vary with alarm evaluation lag and instance warmup.

---

## Corrections Made to the Original Draft

1. **Alarm-before-policy ordering** — the original implied the CloudWatch alarm gets created from inside the ASG's scaling-policy wizard. It doesn't; you create the alarm in CloudWatch first, then reference it by name when building the step scaling policy. Followed literally, the original order leaves you stuck with nothing in the "choose your alarm" dropdown.
2. **Instance warmup field** — missing entirely from the original. Without it, a wave of newly-launched (near-idle) instances can drag down the aggregate CPU metric and trigger an unwanted scale-in shortly after a scale-out.
3. **Console-vs-CLI step boundary semantics** — the original stated CPU ranges as if they're universally absolute. In the console they display as absolute values but are stored as offsets relative to the alarm threshold; CLI/IaC tooling requires you to supply the relative form directly. Worth knowing if this lab ever gets rebuilt in Terraform.
4. **Overlapping stress-ng runs (Part 11)** — the draft told you to launch a second, higher-load `stress-ng` run without stopping the first 10-minute job. That leaves two workloads competing on 2 vCPUs, so the CPU reading you'd see doesn't correspond to the step you're supposedly testing. Fixed by adding a `pkill stress-ng` before the second run.
5. **Production-hardening pass on Part 4** (launch template + user-data), made when you asked for a production-level setup:
   - `HttpTokens: required` enforced on the launch template itself, not just handled ad hoc in the script — blocks IMDSv1-style metadata calls from anything on the box, not only your own curl line.
   - User-data script rewritten with `set -euo pipefail`, logging to `/var/log/user-data.log`, an apt-get retry loop, and an explicit `exit 1` if nginx never actually comes up — the original would boot "successfully" even if nginx failed to install, and you'd be debugging the wrong layer.
   - Instance ID and private IP moved off the public `/` page to `/debug/` — the original published internal AWS identifiers on the ALB's public root page, which is a real recon leak, not a style nit.
   - SSH inbound rule removed entirely from Part 2; access is via Session Manager only (needs `AmazonSSMManagedInstanceCore`, already in Part 3). The original's "SSH from My IP" is the kind of rule that outlives the lab it was scoped for.

**Open item, unresolved:** `CloudWatchAgentServerPolicy` is still attached to the IAM role in Part 3 even though the doc itself says it isn't needed for basic CPU-based step scaling. That's unused permission scope sitting on a "production" role — keep it only if you're actually planning to extend into custom/memory metrics; otherwise it should come off.
