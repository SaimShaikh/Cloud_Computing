
# AWS Systems Manager (SSM) — Complete Deep Dive Guide

> Scenario-driven, production-grade reference covering AWS Systems Manager architecture, internals, Session Manager, Run Command, Patch Manager, Automation, Parameter Store, hybrid/on-prem management, security posture, troubleshooting, and interview preparation.

---



## 1. Scenario

You are the DevOps/Cloud engineer for a company running a mixed fleet:

- 500 Ubuntu EC2 instances across two regions (Mumbai, Singapore)
- 200 Windows Server EC2 instances
- 30 on-premises Linux servers in a data center that has no direct AWS connectivity requirement beyond outbound internet

Requirements from leadership:

- No one should SSH or RDP directly into production servers — every access must be logged and IAM-controlled.
- Every server must be patched every Sunday at 2:00 AM without manual login.
- Secrets (DB passwords, API keys) must not be hardcoded in AMIs or user-data scripts.
- On-prem servers must be visible in the same dashboard as EC2 instances.
- Auditors need a compliance report showing which servers are patched and which are not.

This is exactly the problem AWS Systems Manager (SSM) is built to solve, and this guide walks through every component needed to satisfy that scenario.

---

## 2. What is AWS Systems Manager?

AWS Systems Manager (SSM) is an operations hub that lets you view, manage, automate, and secure infrastructure — EC2 instances, on-premises servers, and virtual machines in a hybrid environment — from a single console, without needing direct network-level access (SSH/RDP) to each machine individually.

It supports:

- EC2 Instances (Linux and Windows)
- On-Premises Servers (physical or virtual, via Hybrid Activations)
- Virtual Machines in other clouds (treated as "hybrid" nodes)

The defining architectural principle: **the managed node always initiates the connection to AWS**, not the other way around. This single fact explains almost every security and networking behavior of SSM.

---

## 3. Why Systems Manager Exists — The Problem It Solves

### Without Systems Manager

```
Administrator
      │
 SSH (22) / RDP (3389)
      │
      ▼
Every EC2 Instance
```

Problems this creates at scale:

- Port 22/3389 must stay open somewhere — a permanent attack surface.
- Bastion hosts become single points of failure and still need patching/hardening themselves.
- SSH key sprawl: who has which key, and how do you revoke one person's access without rotating keys fleet-wide?
- No native audit trail of exactly what commands were run in a session.
- Managing 500+ servers means either scripting SSH in a loop (fragile) or buying a third-party fleet-management tool.

### With Systems Manager

```
Administrator
      │
AWS Console / CLI / SDK
      │
AWS Systems Manager (control plane)
      │
   HTTPS (443) — outbound from the instance
      │
   SSM Agent (running inside the instance)
      │
   EC2 Instance / On-Prem Server
```

Benefits:

- No inbound SSH or RDP port needs to be open.
- Access is controlled entirely through IAM policies — revoke access by editing/removing an IAM policy, not rotating keys.
- Every Session Manager session and Run Command execution can be logged to CloudWatch Logs and/or S3, and correlated in CloudTrail.
- One console (or one CLI call) can target 1, 100, or 10,000 instances via resource groups or tags.

---

## 4. Core Components (Full Map)

| Category | Component | Purpose |
|---|---|---|
| Node Management | Fleet Manager | Unified inventory/dashboard of all managed nodes |
| Node Management | Session Manager | Shell/PowerShell access without SSH/RDP |
| Node Management | Run Command | Execute ad-hoc or scripted commands remotely |
| Node Management | Patch Manager | Detect, approve, and install OS patches |
| Node Management | Inventory | Collects metadata (installed apps, network config, files) |
| Change Management | Automation | Runs multi-step runbooks (SSM Documents) |
| Change Management | Maintenance Windows | Schedules disruptive tasks in a controlled time slot |
| Change Management | Change Manager | Approval workflow gate before changes execute |
| Application Management | Parameter Store | Secure hierarchical config/secret storage |
| Application Management | AppConfig | Dynamic feature flag / config rollout with validation |
| Application Management | Application Manager | Groups resources by application (OpsCenter integration) |
| Operations Management | Explorer | Cross-account/cross-region operational dashboard |
| Operations Management | OpsCenter | Ticket-like "OpsItems" for operational issues |
| Operations Management | Incident Manager | Structured incident response (paging, runbooks, timeline) |
| Shared Services | State Manager | Enforces a desired configuration state on a schedule |
| Shared Services | Distributor | Packages and distributes software/agents across the fleet |

---

## 5. How Systems Manager Works — Internal Architecture

Communication is always initiated from the managed node outward.

```
Managed Node (EC2 / On-Prem)
        │
   SSM Agent (background service)
        │
   Outbound HTTPS request (TCP 443)
        │
        ▼
AWS Systems Manager Service Endpoints
  ssm.<region>.amazonaws.com
  ssmmessages.<region>.amazonaws.com
  ec2messages.<region>.amazonaws.com
```

Three distinct endpoints matter, and mixing them up is a common misconfiguration:

- `ssm.<region>.amazonaws.com` — core SSM API calls (registration, commands, inventory sync)
- `ec2messages.<region>.amazonaws.com` — used by EC2 instances for Run Command / Automation message exchange
- `ssmmessages.<region>.amazonaws.com` — used specifically for Session Manager (control channel + data channel)

If you are using VPC Interface Endpoints for a fully private subnet, **all three** must be created, or specific features silently fail (e.g., Session Manager connects but Run Command doesn't, or vice versa).

AWS's control plane never opens a connection into your VPC. It only ever responds to connections the SSM Agent initiates. This is why Systems Manager works even in subnets with zero inbound rules, as long as outbound HTTPS is allowed.

---

## 6. The SSM Agent — Deep Dive

The SSM Agent is a background process (a Go binary running as a systemd service on Linux, or a Windows service on Windows) that:

- Registers the instance with the Systems Manager service
- Polls/listens for commands, automation steps, and session requests
- Executes commands locally using the OS's native shell or package manager
- Reports status, inventory, and compliance data back to AWS
- Manages the actual terminal session for Session Manager

### Agent presence by AMI

- Amazon Linux 2/2023, most current Ubuntu AMIs, Windows Server AMIs published by AWS: **pre-installed**
- Custom AMIs, older AMIs, or minimal/hardened images: **may need manual installation**

### Registration payload (conceptual)

When the agent starts, it reports details such as:

```json
{
  "InstanceId": "i-0123456789abcdef0",
  "Platform": "Linux",
  "PlatformName": "Ubuntu",
  "PlatformVersion": "24.04",
  "Architecture": "x86_64",
  "AgentVersion": "3.3.x"
}
```

AWS does not guess the OS — the agent tells AWS explicitly. This is why an instance with a stopped or crashed agent shows as "Connection Lost" rather than being reclassified.

### Checking agent status (Linux)

```bash
sudo systemctl status amazon-ssm-agent
sudo systemctl restart amazon-ssm-agent
sudo journalctl -u amazon-ssm-agent -f
```

### Checking agent status (Windows, PowerShell)

```powershell
Get-Service AmazonSSMAgent
Restart-Service AmazonSSMAgent
```

---

## 7. Managed Instances — Requirements and Registration

An instance becomes a "Managed Instance" only when **all** of the following are true:

1. SSM Agent is installed and running
2. An IAM role with the `AmazonSSMManagedInstanceCore` policy (or an equivalent least-privilege custom policy) is attached
3. The instance can reach the required SSM endpoints (via internet, NAT Gateway, or VPC Interface Endpoints)
4. The Security Group allows outbound HTTPS (443) — no inbound rule is required for SSM itself

If any one of these is missing, the instance will not appear as "Managed" in Fleet Manager, even though the EC2 instance itself is running fine. This is the single most common day-one troubleshooting scenario for engineers new to SSM.

---

## 8. Session Manager — Deep Dive

Session Manager provides a browser-based or CLI-based interactive shell to an instance without SSH keys or an open port 22/3389.

### Traditional vs Session Manager

```
Traditional:
Laptop --SSH(22)--> EC2

Session Manager:
Laptop --HTTPS--> AWS Systems Manager --HTTPS--> SSM Agent --> EC2
```

### Key advantages

- No bastion host needed, no key management
- Access governed entirely by IAM policy (`ssm:StartSession`, resource tags, etc.)
- Every keystroke and session can be logged to CloudWatch Logs and/or an S3 bucket for audit
- Sessions can be restricted to specific ports for port-forwarding use cases (e.g., tunneling to an RDS instance in a private subnet without a bastion)

### Use cases

- Emergency access to a fully private subnet instance (no public IP, no NAT) via VPC Interface Endpoints
- Temporary contractor access — grant an IAM policy scoped to a specific tag, revoke it when the contract ends, with zero key rotation
- Port forwarding to reach a private RDS/ElastiCache endpoint from a local laptop for debugging, using the `AWS-StartPortForwardingSessionToRemoteHost` session document to tunnel a local port to a remote host/port through the instance

### Enforcing Session Manager as the only access path

You can attach an IAM policy `Deny` on `ec2:*` SSH-related actions is not how you block SSH — instead you control it by:

- Not attaching a key pair, or restricting Security Group inbound rules to remove port 22/3389 entirely
- Using an SCP (Service Control Policy) at the AWS Organizations level to deny launching instances with inbound 22/3389 open to `0.0.0.0/0`

---

## 9. Run Command — Deep Dive

Run Command executes a predefined or ad-hoc command/document across one, many, or thousands of instances, without any persistent session.

### Example commands by OS

Ubuntu/Debian:
```bash
sudo apt update && sudo apt upgrade -y
```

Amazon Linux/RHEL:
```bash
sudo dnf update -y
```

Windows (PowerShell):
```powershell
Get-Service | Where-Object {$_.Status -eq 'Stopped'}
```

### Targeting

Run Command can target instances by:

- Instance ID (explicit list)
- Tag (e.g., `Environment=Production`, `App=WebServer`)
- Resource Group

### Use case: fleet-wide emergency action

If a CVE is announced for a specific package, you can run:

```bash
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters 'commands=["sudo apt-get install --only-upgrade openssl -y"]' \
  --comment "Emergency OpenSSL patch"
```

...against every production Linux instance simultaneously, and get per-instance success/failure output back in the console — no scripting a loop of SSH connections.

### Output handling

Run Command output can be sent to:

- S3 (long-term audit trail)
- CloudWatch Logs (real-time monitoring/alerting)
- Directly viewable in the console (last output only, size-limited)

---

## 10. Patch Manager — Deep Dive

Patch Manager automates OS-level patching using **patch baselines**, which define which patches are auto-approved and after how many days.

### Key concepts

- **Patch Baseline** — rules for auto-approval (e.g., "auto-approve Critical and Important security patches 7 days after release")
- **Patch Group** — a tag (`Patch Group = ProductionWeb`) used to associate instances with a specific baseline
- **Patch Compliance** — the reported state (Compliant / Non-Compliant / Missing / Failed) per instance

### What Patch Manager does NOT do

- It does **not** maintain its own list of Linux/Windows updates.
- It does **not** build patches.
- It **orchestrates** the OS's own native package manager to do the actual work.

This distinction matters for interviews and for real troubleshooting — a "failed patch" is very often actually a repository connectivity issue (e.g., `archive.ubuntu.com` unreachable due to a missing NAT Gateway route) rather than an SSM issue.

---

## 11. How Patch Manager Detects and Installs Updates (Step-by-Step)

```
Step 1: AWS triggers Patch Manager (scheduled or manual)
             │
             ▼
Step 2: SSM Agent identifies OS type (Ubuntu / Amazon Linux / Windows)
             │
             ▼
Step 3: SSM Agent invokes the native package manager
   Ubuntu:        apt update && apt list --upgradable
   Amazon Linux:  dnf check-update
   Windows:       Windows Update API
             │
             ▼
Step 4: OS contacts its official repository
   Ubuntu:        archive.ubuntu.com / security.ubuntu.com
   Amazon Linux:  Amazon Linux repository
   Windows:       Microsoft Windows Update
             │
             ▼
Step 5: Repository responds with available versions
   e.g. OpenSSH installed=9.0.0, latest=9.0.1 → Missing Patch
             │
             ▼
Step 6: SSM Agent reports result to AWS
   AWS marks instance: Non-Compliant
             │
             ▼
Step 7: If within a Maintenance Window / Install operation:
   AWS instructs SSM Agent → "Install Approved Patches"
   Ubuntu:        apt upgrade -y
   Amazon Linux:  dnf update -y
   Windows:       Windows Update API applies updates
             │
             ▼
Step 8: SSM Agent reports back → AWS marks instance: Compliant
```

### Scan vs Install operations

- **Scan** — read-only, only reports what is missing; safe to run anytime.
- **Install** — actually applies updates and may trigger a reboot; should always run inside a Maintenance Window.

---

## 12. Maintenance Windows

A Maintenance Window defines a recurring time slot during which disruptive tasks (patching, reboots, backups) are permitted to run.

### Example

```
Schedule: Every Sunday, 02:00 AM, duration 4 hours
   │
   ▼
Task 1: Patch Scan
   │
   ▼
Task 2: Patch Install
   │
   ▼
Task 3: Reboot (if required)
   │
   ▼
Task 4: Send Compliance Report (SNS / CloudWatch)
```

Maintenance Windows also support a **cutoff** setting — tasks that haven't started within the cutoff period before the window ends will not be started, preventing a patch install from beginning right as the window is about to close.

---

## 13. Automation (Runbooks)

Automation runs multi-step workflows defined as SSM Documents (JSON/YAML), useful for repeatable operational procedures.

### Example runbook: safe EBS volume resize

```
Step 1: Stop EC2 instance
   │
   ▼
Step 2: Create EBS snapshot (rollback point)
   │
   ▼
Step 3: Modify volume size
   │
   ▼
Step 4: Start EC2 instance
   │
   ▼
Step 5: Verify instance status checks pass
```

### Why this matters operationally

Automation documents remove "tribal knowledge" — a junior engineer can safely execute a documented, tested runbook instead of manually running commands from memory at 2 AM during an incident. Automation can also be triggered by EventBridge rules (e.g., auto-remediate a specific CloudWatch alarm), a pattern commonly used in AIOps/auto-remediation pipelines.

---

## 14. Parameter Store — Deep Dive

Parameter Store is a hierarchical key-value store for configuration data and secrets.

### Parameter types

| Type | Use case | Encryption |
|---|---|---|
| String | Non-sensitive config (e.g., feature flag, region name) | None |
| StringList | Comma-separated config values | None |
| SecureString | Passwords, API keys, tokens | KMS (AWS-managed or CMK) |

### Hierarchical naming (best practice)

```
/prod/webapp/db/password
/prod/webapp/db/host
/staging/webapp/db/password
```

This structure lets you use IAM path-based policies (e.g., allow read access only to `/staging/*`) to segregate environments cleanly.

### Application integration pattern

Applications retrieve values at startup or runtime (via SDK, or via the SSM Agent's local `ssm-parameter-store` integration for ECS/EKS), removing the need to bake secrets into AMIs, Docker images, or user-data scripts.

---

## 15. Parameter Store vs Secrets Manager

| Feature | Parameter Store | Secrets Manager |
|---|---|---|
| Cost | Standard tier free; Advanced tier has a small per-parameter fee | Charged per secret per month + API calls |
| Automatic rotation | Not built-in (requires custom Lambda + State Manager/Automation) | Built-in rotation for RDS, DocumentDB, Redshift, and custom Lambda rotation |
| Max value size | 4 KB (Standard), 8 KB (Advanced) | 64 KB |
| Cross-account sharing | Limited | Native resource policies |
| Best fit | General app config + simple secrets, cost-sensitive workloads | Secrets needing automatic rotation (DB credentials) |

**Practical guidance:** many teams use Parameter Store for configuration and static/rarely-rotated secrets, and reserve Secrets Manager specifically for database credentials that need automated rotation.

---

## 16. State Manager, Inventory, Distributor, Fleet Manager

- **State Manager** — continuously enforces a desired configuration (e.g., "antivirus agent must always be running"; if State Manager detects drift, it re-applies the association on its schedule).
- **Inventory** — collects metadata: installed applications, network configuration, Windows registry values, running services — useful for license audits and security posture checks.
- **Distributor** — packages and pushes software packages (including third-party agents like Datadog or CrowdStrike) to the fleet in a controlled, versioned way.
- **Fleet Manager** — the unified visual dashboard showing every managed node, its compliance state, and quick actions (start session, view logs, reboot) in one place.

---

## 17. Hybrid Activations — Managing On-Premises Servers

Systems Manager isn't limited to EC2. On-premises or other-cloud VMs can be registered as **Managed Instances** using a **Hybrid Activation**:

```
Step 1: Create an Activation in SSM Console
        (generates an Activation Code + Activation ID, expiry, and an IAM role)
             │
             ▼
Step 2: Install SSM Agent on the on-prem server manually
             │
             ▼
Step 3: Register the server using the Activation Code/ID
   sudo amazon-ssm-agent -register \
     -code "<activation-code>" \
     -id "<activation-id>" \
     -region "ap-south-1"
             │
             ▼
Step 4: Server appears in Fleet Manager as a Managed Instance
        (prefixed "mi-" instead of "i-")
```

This directly satisfies the scenario requirement of viewing on-prem and EC2 servers in one dashboard — no VPN or Direct Connect is required purely for SSM to function, only outbound internet HTTPS access from the on-prem server.

---

## 18. Networking Requirements — Internet vs VPC Endpoints

| Scenario | Requirement |
|---|---|
| Public subnet with internet access | Outbound HTTPS (443) allowed in Security Group/NACL |
| Private subnet with NAT Gateway | NAT Gateway route to internet; outbound 443 allowed |
| Fully private subnet, no NAT/IGW | VPC Interface Endpoints for `ssm`, `ssmmessages`, `ec2messages` (and optionally `kms` if using SecureString parameters, and `logs`/`s3` if streaming session logs) |

**Common mistake:** creating only the `ssm` endpoint and assuming Session Manager will work. Session Manager specifically requires the `ssmmessages` endpoint; without it, Run Command may succeed while Session Manager connections time out.

---

## 19. Console-First Hands-On Steps

### A. Enable an EC2 instance for SSM

1. Launch (or select) an EC2 instance.
2. Attach an IAM role containing the `AmazonSSMManagedInstanceCore` managed policy.
3. Confirm the Security Group allows outbound HTTPS (443).
4. Wait 1–2 minutes, then go to **Systems Manager → Fleet Manager** — the instance should appear with **Ping Status: Online**.

### B. Start a Session Manager session

1. Go to **Systems Manager → Session Manager → Start session**.
2. Select the target instance.
3. Click **Start session** — a browser-based terminal opens directly, no key pair needed.

### C. Run a command across a fleet

1. Go to **Systems Manager → Run Command → Run command**.
2. Choose document `AWS-RunShellScript` (Linux) or `AWS-RunPowerShellScript` (Windows).
3. Under **Targets**, select "Specify instance tags" and enter `Environment=Production`.
4. Enter the command, choose an S3 bucket for output logging, and click **Run**.

### D. Set up Patch Manager

1. Go to **Systems Manager → Patch Manager → Configure patching**.
2. Choose or create a **Patch Baseline** (or use the AWS-provided default).
3. Tag target instances with a **Patch Group** (e.g., `Patch Group = ProductionWeb`).
4. Create a **Maintenance Window**, attach the `AWS-RunPatchBaseline` task, and schedule it (e.g., cron `cron(0 2 ? * SUN *)`).
5. After the window runs, check **Patch Manager → Compliance** for a per-instance report.

---

## 20. Real-World Use Cases

### Use Case 1: Bastion-less access to fully private subnets
An engineering team removes all bastion hosts and public IPs from production. VPC Interface Endpoints for `ssm`, `ssmmessages`, and `ec2messages` are created, and all access happens through Session Manager — cutting both attack surface and bastion maintenance overhead to zero.

### Use Case 2: Fleet-wide zero-downtime patching
A 500-server fleet is patched every Sunday at 2 AM using Patch Manager + Maintenance Windows, with `max-concurrency` set to 25% so only a quarter of the fleet reboots at a time, keeping the service available throughout the window.

### Use Case 3: Emergency CVE response
When a critical CVE is disclosed, Run Command pushes an emergency package upgrade to every instance tagged `Environment=Production` within minutes, with per-instance pass/fail output captured to S3 for the security team's audit trail.

### Use Case 4: Secrets removed from AMIs and user-data
Instead of baking DB credentials into a golden AMI or passing them via EC2 user-data (which is visible via the instance metadata service to anyone with local access), applications fetch `SecureString` parameters from Parameter Store at boot time, scoped by IAM path-based policy per environment.

### Use Case 5: Hybrid on-prem visibility for auditors
30 on-premises Linux servers are registered via Hybrid Activation. Auditors get one Fleet Manager dashboard showing patch compliance across both cloud and on-prem infrastructure, instead of two separate systems.

### Use Case 6: Auto-remediation (AIOps pattern)
An EventBridge rule watches for a CloudWatch alarm (e.g., disk usage > 90%). When triggered, it invokes an SSM Automation runbook that clears temp files and extends the volume — remediating without paging an engineer at night.

---

## 21. Edge Cases and Failure Scenarios

| Scenario | Root Cause | Resolution |
|---|---|---|
| Instance never appears in Fleet Manager | Missing IAM role, or role missing `AmazonSSMManagedInstanceCore` | Attach the correct IAM role/instance profile; wait for next agent check-in |
| Instance shows "Connection Lost" | SSM Agent crashed, or outbound 443 blocked by Security Group/NACL | Restart agent via EC2 user-data or console; verify SG/NACL egress rules |
| Session Manager fails but Run Command works | Missing `ssmmessages` VPC endpoint | Create the `ssmmessages` Interface Endpoint alongside `ssm` and `ec2messages` |
| Patch install reports "Failed" | Underlying `apt`/`dnf` couldn't reach the OS repository (NAT Gateway misconfigured, or repo mirror down) | Test connectivity manually (`apt update`) via Session Manager; fix routing before re-running |
| SecureString parameter returns encrypted gibberish | Called `get-parameter` without `--with-decryption` | Always pass `--with-decryption` for SecureString reads |
| Maintenance Window task never runs | No targets registered, or targets don't match the tag filter | Verify `aws_ssm_maintenance_window_target` tag matches the actual instance tag exactly (case-sensitive) |
| Hybrid on-prem server won't register | Activation Code/ID expired (default 24 hours unless extended) | Regenerate the Activation with a longer expiry window |
| Run Command "TimedOut" on some instances | Instance under heavy load, or agent temporarily unresponsive | Increase the command timeout parameter; investigate agent health separately |

---

## 22. Troubleshooting Table

| Symptom | Check | Fix |
|---|---|---|
| Ping Status = Connection Lost | `sudo systemctl status amazon-ssm-agent` | Restart the agent; check `/var/log/amazon/ssm/errors.log` |
| Instance not listed at all | IAM role attached? | Attach instance profile with `AmazonSSMManagedInstanceCore` |
| Session opens then instantly closes | Check IAM permissions (`ssm:StartSession` on the resource) | Update IAM policy to allow the action on the target instance tag/ARN |
| Patch compliance stuck at "Missing" | Was an Install operation ever run, or only Scan? | Ensure the Maintenance Window task uses `Operation = Install`, not `Scan` |
| Parameter Store access denied from app | IAM role of the app lacks `ssm:GetParameter` / `kms:Decrypt` | Add both actions — SecureString needs KMS decrypt permission too |

---

## 23. Security and Compliance Notes

- Session Manager sessions can be configured to log every command to CloudWatch Logs and/or S3 — required for most SOC 2 / ISO 27001 audit trails.
- IAM policies can restrict Session Manager access to specific instance tags, specific ports (for port forwarding), or even disable the ability to run shell commands entirely (document-only sessions).
- KMS customer-managed keys (CMKs) can be used for SecureString parameters instead of the AWS-managed key, giving fine-grained key policy control and CloudTrail visibility into every decrypt call.
- Patch compliance data feeds directly into AWS Config and Security Hub for centralized compliance dashboards.
- `AmazonSSMManagedInstanceCore` is broad; for stricter least-privilege environments, a custom policy can be scoped down (e.g., removing `ssm:GetParameter` fleet-wide if not needed).

---

## 24. Interview Questions

**Q1. What is AWS Systems Manager?**
A centralized AWS service for securely managing, monitoring, automating, and patching EC2, on-premises, and hybrid infrastructure without requiring direct SSH/RDP access.

**Q2. Why doesn't Session Manager require SSH?**
Because the SSM Agent initiates an outbound HTTPS connection to AWS; AWS never opens an inbound connection to the instance.

**Q3. How does Patch Manager know what updates are available?**
It does not maintain its own patch database. It instructs the SSM Agent to query the OS's native package manager (`apt`, `dnf`, or Windows Update), which contacts the vendor's own repository.

**Q4. What's the difference between Scan and Install patch operations?**
Scan only reports compliance status; Install actually applies the patches and may trigger a reboot.

**Q5. What IAM policy is minimally required for an EC2 instance to become SSM-managed?**
`AmazonSSMManagedInstanceCore`.

**Q6. Why might Session Manager fail while Run Command succeeds in a private subnet?**
The `ssmmessages` VPC Interface Endpoint is missing — Session Manager specifically depends on it, separate from `ssm` and `ec2messages`.

**Q7. How do you manage on-premises servers with Systems Manager?**
Through a Hybrid Activation: generate an Activation Code/ID, install the SSM Agent on the server, and register it — it then appears as a Managed Instance (`mi-` prefix) alongside EC2 instances.

**Q8. When would you choose Secrets Manager over Parameter Store?**
When you need automatic credential rotation (e.g., RDS database passwords) — Parameter Store has no native rotation and requires a custom Lambda + scheduling solution.

**Q9. What triggers an instance to show as "Non-Compliant" in Patch Manager?**
A patch scan detects that an approved patch (per the patch baseline) is missing from the instance.

**Q10. How can Automation documents support an AIOps auto-remediation pipeline?**
An EventBridge rule can trigger an SSM Automation runbook directly in response to a CloudWatch alarm, executing a pre-tested remediation sequence without human intervention.

---

## 25. Cheat Sheet

```
No inbound SSH/RDP needed        → Session Manager
Run command on many instances    → Run Command
Automate OS patching              → Patch Manager + Maintenance Windows
Multi-step operational workflow   → Automation (SSM Documents)
Store secrets/config securely     → Parameter Store (SecureString)
Rotate DB credentials automatically → Secrets Manager
Manage on-prem servers             → Hybrid Activation
Fully private subnet, no NAT      → VPC Interface Endpoints: ssm, ssmmessages, ec2messages
Unified fleet dashboard            → Fleet Manager
Continuously enforce config drift  → State Manager
Collect installed app/OS metadata  → Inventory
Push third-party agents/software   → Distributor
```

---

## 26. Mastery Checklist

- [ ] Can explain why the managed node initiates the connection, not AWS
- [ ] Can list all three SSM VPC endpoints and what each is for
- [ ] Can start a Session Manager session via the console
- [ ] Can run a command across a tagged fleet via Run Command
- [ ] Can create a custom Patch Baseline and Patch Group
- [ ] Can schedule a Maintenance Window with cron syntax
- [ ] Can store and retrieve a SecureString parameter with KMS decryption
- [ ] Can explain Parameter Store vs Secrets Manager trade-offs
- [ ] Can register an on-premises server via Hybrid Activation
- [ ] Can diagnose "Connection Lost" and "Session Manager fails, Run Command works" scenarios
- [ ] Can describe an EventBridge → Automation auto-remediation pattern

---

## 27. Cleanup Steps

Perform in this order to avoid dependency errors:

1. Deregister any Hybrid Activations no longer needed
2. Delete Maintenance Window tasks, then targets, then the Maintenance Window itself
3. Delete custom Patch Baselines and Patch Groups (detach the patch group from the baseline first)
4. Delete Parameter Store parameters
5. Detach and delete the IAM instance profile, then the IAM role and its policy attachments
6. Delete any VPC Interface Endpoints created solely for SSM testing (`ssm`, `ssmmessages`, `ec2messages`)
7. Terminate any test EC2 instances used for this lab

---

## Conclusion

AWS Systems Manager consolidates fleet management, patching, secure remote access, secrets management, and automation into a single control plane built around one core principle: managed nodes always call home to AWS, never the reverse. Mastering the three endpoint types, the patch scan/install lifecycle, and the IAM-driven access model covers the vast majority of both real-world operational needs and interview questions on this service.
