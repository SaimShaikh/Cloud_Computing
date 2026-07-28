# AWS Inspector 


## Before You Start

- [ ] You are logged into the AWS Console
- [ ] Region (top-right corner) is set to **Asia Pacific (Mumbai) ap-south-1**
- [ ] You have Docker installed on your laptop (`docker --version` works in terminal)
- [ ] You have AWS CLI installed (`aws --version` works in terminal)
- [ ] You have run `aws configure` at least once so CLI can talk to your account

---

# PART 1 — Turn On AWS Inspector

### Step 1
In the AWS Console search bar (top), type **Inspector** and click on **Amazon Inspector**.

### Step 2
You'll land on the Inspector welcome page. Click the button **Get started** (or **Activate Inspector**).

### Step 3
On the activation screen, you'll see three checkboxes/toggles:
- Amazon EC2 scanning
- Amazon ECR scanning
- AWS Lambda scanning

Make sure **all three are turned ON / checked**.

### Step 4
Click **Activate Inspector** at the bottom of the page.

### Step 5
Wait. A loading message will appear. This takes **2–3 minutes**. Don't refresh aggressively — just wait.

### Step 6
Once it's done, click on **Account management** in the left-hand menu.

### Step 7
Check that all three rows (EC2, ECR, Lambda) show status **Enabled**. If any shows "Disabling" or blank, wait 1 more minute and refresh the page.

✅ **Checkpoint:** If all three say "Enabled," Part 1 is done. Move to Part 2.

---

# PART 2 — EC2 Lab (Vulnerable Server)

## 2A. Create an IAM Role (so EC2 can talk to Systems Manager)

### Step 8
In the search bar, type **IAM** and click **IAM**.

### Step 9
In the left menu, click **Roles**.

### Step 10
Click the orange **Create role** button.

### Step 11
Under "Trusted entity type," make sure **AWS service** is selected.

### Step 12
Under "Use case," select **EC2** from the list. Click **Next**.

### Step 13
In the search box for permissions policies, type **AmazonSSMManagedInstanceCore**.

### Step 14
Check the checkbox next to **AmazonSSMManagedInstanceCore**. Click **Next**.

### Step 15
Give the role a name: `EC2-SSM-Inspector-Role`

### Step 16
Scroll down and click **Create role**.

✅ **Checkpoint:** You should see a green success banner: "Role EC2-SSM-Inspector-Role created."

---

## 2B. Launch the EC2 Instance

### Step 17
In the search bar, type **EC2** and click **EC2**.

### Step 18
Click the orange **Launch instance** button.

### Step 19
Under "Name," type: `inspector-lab-ec2`

### Step 20
Under "Application and OS Images (Amazon Machine Image)," in the search box, type **Amazon Linux 2** and select **Amazon Linux 2 AMI** (NOT Amazon Linux 2023 — we specifically want the older one for this lab).

### Step 21
Under "Instance type," select **t3.micro**.

### Step 22
Under "Key pair (login)," either:
- Select an existing key pair you already have, **or**
- Click **Create new key pair**, name it `inspector-lab-key`, click **Create key pair** (this downloads a `.pem` file — save it somewhere safe, you'll need it in Step 27).

### Step 23
Under "Network settings," click **Edit**.

### Step 24
Confirm **Auto-assign public IP** is set to **Enable**.

### Step 25
Under Security Group settings, check **Allow SSH traffic from**, and set the source to **Anywhere (0.0.0.0/0)** — not "My IP."

> **Why not "My IP"?** We'll connect using the browser-based **EC2 Instance Connect** (Step 33-34), which reaches your instance from AWS's own service IP range, not your personal IP. If the rule is locked to "My IP," that connection will fail. `0.0.0.0/0` is fine here only because this is a short-lived, throwaway lab instance that gets terminated in Part 7 — never leave a real production instance open like this.

### Step 26
Scroll down to **Advanced details**. Find **IAM instance profile** and select **EC2-SSM-Inspector-Role** (the role you created in Step 15).

### Step 27
Scroll to the bottom and click the orange **Launch instance** button.

### Step 28
Click **View all instances** to go back to the EC2 instance list.

### Step 29
Wait until the **Instance state** column shows **Running** and **Status check** shows **2/2 checks passed** (this takes 2–3 minutes — refresh the page if needed).

✅ **Checkpoint:** Instance is Running with 2/2 status checks passed.

---

## 2C. Confirm the Instance Registered with Systems Manager

### Step 30
In the search bar, type **Systems Manager** and click it.

### Step 31
In the left menu, click **Fleet Manager**.

### Step 32
Look for `inspector-lab-ec2` in the list. Its **Ping status** should show **Online**.

> **If it's not there after 5 minutes:** go back to EC2 → select your instance → Actions → Security → Modify IAM role → confirm `EC2-SSM-Inspector-Role` is attached. Wait a few more minutes and check again.

✅ **Checkpoint:** Instance shows Online in Fleet Manager. This means Inspector will now be able to scan it.

<img width="3247" height="1185" alt="image" src="https://github.com/user-attachments/assets/9d5ac84f-1516-43c8-bc13-701c319cde8d" />

---

## 2C-1. SSM Networking Requirements — VPC Endpoints & Security Groups Explained

This lab intentionally uses the simplest setup, so it's worth being explicit about *why* it works and when it wouldn't.

### Do we need VPC Endpoints here?
**No — not in this lab.** Our instance:
- Has a **public IP** (Step 24)
- Sits in the **default VPC**, which already routes 0.0.0.0/0 to an **Internet Gateway**

The SSM Agent reaches the AWS Systems Manager service over the public internet via that Internet Gateway. No VPC endpoint is required.

**When you WOULD need VPC endpoints:** if the instance sits in a **private subnet with no route to the internet** (no IGW, no NAT Gateway) — same scenario as an earlier private-subnet SSM lab. In that case you'd need three **Interface VPC Endpoints** in that subnet:
- `com.amazonaws.<region>.ssm`
- `com.amazonaws.<region>.ssmmessages`
- `com.amazonaws.<region>.ec2messages`

Each of those endpoints also needs its **own security group** allowing **inbound HTTPS (443)** from the EC2 instance's security group (or its subnet CIDR) — otherwise the instance can reach the endpoint's ENI but the connection gets dropped.

### Do we need a security group rule for SSM itself?
**No inbound rule needed.** The SSM Agent only initiates **outbound** connections (agent → SSM service on port 443). It never listens for inbound traffic. Default security groups allow all outbound traffic by default, so this "just works" with zero extra configuration.

The **only** inbound rule this lab actually needs is the port 22 rule for EC2 Instance Connect (already set in Step 25) — that's unrelated to SSM/Inspector and purely so you can log in and intentionally leave the box unpatched.

### Quick reference

| Scenario | VPC Endpoints Needed? | Security Group Rule Needed? |
|---|---|---|
| Public subnet + IGW (this lab) | ❌ No | Outbound: default (already open). No inbound rule needed for SSM. |
| Private subnet, no internet route | ✅ Yes — `ssm`, `ssmmessages`, `ec2messages` interface endpoints | Endpoint SG: inbound 443 from instance SG/CIDR. Instance SG: outbound 443 (default is fine). |

---

## 2D. Connect to the Instance and Make It Vulnerable (Don't Patch It)

### Step 33
Go back to **EC2 → Instances**, select `inspector-lab-ec2`, click **Connect** button at the top.

### Step 34
Choose the **EC2 Instance Connect** tab (easiest — works right in the browser, no need for the `.pem` file). Click **Connect**.

### Step 35
A terminal window opens in your browser, already logged into the instance. You're now "inside" the server.

### Step 36
Type this command and press Enter (just to look at the current version — don't change anything yet):
```bash
openssl version
```

### Step 37
Install an extra package so there's more surface area for findings:
```bash
sudo yum install -y httpd
```
<img width="3296" height="1648" alt="image" src="https://github.com/user-attachments/assets/6bf26a51-6f23-46fc-b3f0-180711c8593e" />
<img width="1624" height="607" alt="Screenshot 2026-07-28 at 4 40 45 PM" src="https://github.com/user-attachments/assets/c1b63425-2d71-4d6a-b9a2-c3a6352ea8cc" />


### Step 38
**Important: do NOT run `sudo yum update -y` yet.** We want the instance to stay outdated on purpose so Inspector has real vulnerabilities to find. Just close this browser tab for now.

✅ **Checkpoint:** Part 2D done. You deliberately left the instance unpatched.

---

## 2E. Wait for Inspector to Scan the EC2 Instance

### Step 39
Wait **15–30 minutes**. Go get a coffee — this is normal, real scan timing, not a bug.

### Step 40
In the search bar, type **Inspector** and click it.

### Step 41
In the left menu, click **Findings** → **All findings**.

### Step 42
At the top, use the filter dropdown labeled **Resource type**, choose **EC2 Instance**.

### Step 43
You should now see one or more rows — each one is a real vulnerability (CVE) found on your instance.

### Step 44
Click on any one finding row to open its details.

### Step 45
Read these fields (this is the important learning part):
- **Severity** (Critical / High / Medium / Low)
- **Vulnerability ID** (the CVE number)
- **Affected package** and its installed version vs. fixed version
- **Network reachability** — since your instance has a public IP, this should show as reachable, which raises its priority

✅ **Checkpoint:** You've seen at least one real finding with severity and CVE details.

---

## 2F. Fix the Vulnerability and Confirm It Closes

### Step 46
Go back to **EC2 → Instances → inspector-lab-ec2 → Connect → EC2 Instance Connect → Connect** (same as Steps 33-34).

### Step 47
Now actually patch the system:
```bash
sudo yum update -y
```

### Step 48
Reboot the instance:
```bash
sudo reboot
```

### Step 49
Wait **15–30 minutes** again (Inspector needs to rescan).

### Step 50
Go back to **Inspector → Findings → All findings**, filter by **EC2 Instance** again.

### Step 51
Check the finding(s) you saw earlier — their status should now show **Closed** or the finding should no longer appear in the active list.

✅ **Checkpoint (Part 2 complete):** You found a real EC2 vulnerability and watched it get remediated and closed.

---

# PART 3 — ECR Lab (Vulnerable Docker Image)

## 3A. Create the ECR Repository

### Step 52
In the AWS Console search bar, type **ECR** and click **Elastic Container Registry**.

### Step 53
Click **Create repository**.

### Step 54
Under "Repository name," type: `inspector-lab-repo`

### Step 55
Leave everything else default. Click **Create repository** at the bottom.

✅ **Checkpoint:** Repository `inspector-lab-repo` appears in the repository list.

---

## 3B. Build a Deliberately Vulnerable Docker Image (on your laptop)

### Step 56
On your computer, open a terminal and create a new folder:
```bash
mkdir inspector-lab-docker
cd inspector-lab-docker
```

### Step 57
Create a file named `Dockerfile` (no extension) with this exact content:
```dockerfile
FROM node:14.0.0
WORKDIR /app
COPY package.json .
RUN npm install
COPY . .
CMD ["node", "index.js"]
```

### Step 58
Create a file named `package.json` with this exact content:
```json
{
  "name": "inspector-lab-app",
  "version": "1.0.0",
  "dependencies": {
    "lodash": "4.17.15",
    "express": "4.16.0"
  }
}
```

### Step 59
Create a file named `index.js` with this exact content:
```js
console.log("Inspector lab test app running");
```

> **Why these versions?** `node:14.0.0`, `lodash@4.17.15`, and `express@4.16.0` are old enough to have real, publicly known CVEs — this guarantees Inspector will find something.

✅ **Checkpoint:** You have 3 files in the `inspector-lab-docker` folder: `Dockerfile`, `package.json`, `index.js`.

---

## 3C. Push the Image to ECR

### Step 60
Go back to the AWS Console → **ECR** → click on `inspector-lab-repo`.

### Step 61
Click the **View push commands** button (top right). This gives you 4 exact commands customized for your account — copy them one at a time into your terminal.

### Step 62
Run the **login command** it gives you (looks like this, but use the exact one shown to you):
```bash
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.ap-south-1.amazonaws.com
```
You should see `Login Succeeded`.

### Step 63
Run the **build command** (in your `inspector-lab-docker` folder):
```bash
docker build -t inspector-lab-repo .
```
Wait for it to finish — you'll see several "Step X/X" lines.

### Step 64
Run the **tag command**:
```bash
docker tag inspector-lab-repo:latest <your-account-id>.dkr.ecr.ap-south-1.amazonaws.com/inspector-lab-repo:latest
```

### Step 65
Run the **push command**:
```bash
docker push <your-account-id>.dkr.ecr.ap-south-1.amazonaws.com/inspector-lab-repo:latest
```
Wait for all layers to show "Pushed."

✅ **Checkpoint:** Push completes without errors. Go back to the ECR console, click into `inspector-lab-repo`, and confirm you see an image with tag `latest`.

---

## 3D. View the Findings

### Step 66
Wait **5–10 minutes** (ECR scans are faster than EC2).

### Step 67
In AWS Console, go to **Inspector → Findings → All findings**.

### Step 68
Filter by **Resource type: AMI/Container Image** (or similar wording — pick the container image option).

### Step 69
You should see multiple findings — some for the old Node.js base image, some for `lodash`, some for `express`.

### Step 70
Click one open and read the CVE ID, severity, and fixed version — same as you did for EC2.

✅ **Checkpoint:** You've seen real container image findings.

---

## 3E. Fix the Image and Push Again

### Step 71
Edit your `Dockerfile` — change the first line to:
```dockerfile
FROM node:20-alpine
```

### Step 72
Edit your `package.json` — change the dependencies to:
```json
{
  "dependencies": {
    "lodash": "^4.17.21",
    "express": "^4.19.2"
  }
}
```

### Step 73
Rebuild, tag, and push again (repeat Steps 63, 64, 65 exactly — same commands).

### Step 74
Wait **5–10 minutes**, then go to **ECR → inspector-lab-repo** — you'll now see **two images** (two different digests). Click into the newest one's **Scan results**.

### Step 75
Confirm the new image has few or zero findings compared to the old one.

✅ **Checkpoint (Part 3 complete):** You found real container vulnerabilities, fixed them, and confirmed the new image is cleaner. Note: the OLD image's findings still exist — that's expected, since Inspector tracks findings per image, not per repo.

---

# PART 4 — Lambda Lab (Vulnerable Function)

## 4A. Build the Deployment Package (on your laptop)

### Step 76
Open a terminal and create a new folder:
```bash
mkdir inspector-lab-lambda
cd inspector-lab-lambda
mkdir package
```

### Step 77
Install an old, vulnerable Python package into the `package` folder:
```bash
pip install requests==2.6.0 -t package/
```

### Step 78
Create a file named `lambda_function.py` (in the `inspector-lab-lambda` folder, not inside `package`) with this content:
```python
import requests

def lambda_handler(event, context):
    return {"statusCode": 200, "body": "Inspector Lambda lab running"}
```

### Step 79
Copy that file into the `package` folder too:
```bash
cp lambda_function.py package/
```

### Step 80
Zip everything inside the `package` folder:
```bash
cd package
zip -r ../function.zip .
cd ..
```

✅ **Checkpoint:** You now have a file called `function.zip` in the `inspector-lab-lambda` folder.

---

## 4B. Create the IAM Role for Lambda

### Step 81
In AWS Console, go to **IAM → Roles → Create role**.

### Step 82
Trusted entity type: **AWS service**. Use case: **Lambda**. Click **Next**.

### Step 83
Search for and check **AWSLambdaBasicExecutionRole**. Click **Next**.

### Step 84
Name it: `inspector-lab-lambda-role`. Click **Create role**.

✅ **Checkpoint:** Role created.

---

## 4C. Create the Lambda Function

### Step 85
In AWS Console search bar, type **Lambda** and click it.

### Step 86
Click **Create function**.

### Step 87
Choose **Author from scratch**.

### Step 88
Function name: `inspector-lab-lambda`

### Step 89
Runtime: **Python 3.9**

### Step 90
Under "Permissions," expand **Change default execution role**, choose **Use an existing role**, and select `inspector-lab-lambda-role`.

### Step 91
Click **Create function**.

### Step 92
Once created, scroll down to the **Code** section. Click **Upload from** → **.zip file**.

### Step 93
Click **Upload**, select your `function.zip` file from Step 80.

### Step 94
Click **Save**.

✅ **Checkpoint:** Function shows "Successfully updated" or similar confirmation.

---

## 4D. View the Findings

### Step 95
Wait **15 minutes**.

### Step 96
Go to **Inspector → Findings → All findings**.

### Step 97
Filter by **Resource type: AWS Lambda function**.

### Step 98
Click on the finding — it should reference the `requests` package and its known CVE.

✅ **Checkpoint:** You've seen a real Lambda finding.

---

## 4E. Fix the Function

### Step 99
Back in your terminal, in the `inspector-lab-lambda` folder:
```bash
rm -rf package/requests*
pip install requests==2.32.3 -t package/
cd package
zip -r ../function.zip .
cd ..
```

### Step 100
In the AWS Console, go back to your Lambda function → **Code** → **Upload from** → **.zip file** → upload the new `function.zip` → **Save**.

### Step 101
Wait **15 minutes**, then check **Inspector → Findings** again, filtered to Lambda — the old finding should now be closed/resolved.

✅ **Checkpoint (Part 4 complete):** Lambda vulnerability found and fixed.

---

# PART 5 — Practice Reading & Prioritizing Findings

### Step 102
Go to **Inspector → Findings → All findings** (remove all filters so you see everything from Parts 2, 3, and 4 together).

### Step 103
Click the **Severity** column header (or use the filter) to sort/group so **Critical** and **High** findings are at the top.

### Step 104
For each Critical/High finding, note down: which resource, what CVE, is it internet-reachable.

### Step 105
Write one sentence deciding what you'd fix first and why (for example: "The internet-facing EC2 instance's Critical finding first, since it's both severe and publicly reachable").

✅ **Checkpoint:** You can explain, in your own words, how you'd prioritize a list of mixed findings.

---

# PART 6 (Optional) — Get an Email Alert for Critical Findings

### Step 106
In AWS Console search bar, type **SNS** and click it.

### Step 107
Click **Topics** in the left menu, then **Create topic**.

### Step 108
Type: **Standard**. Name: `inspector-lab-alerts`. Click **Create topic**.

### Step 109
On the topic page, click **Create subscription**.

### Step 110
Protocol: **Email**. Endpoint: type your own email address. Click **Create subscription**.

### Step 111
Check your email inbox — click **Confirm subscription** in the email AWS just sent you.

### Step 112
In AWS Console search bar, type **EventBridge** and click it.

### Step 113
In the left menu, click **Rules**, then **Create rule**.

### Step 114
Name: `inspector-critical-findings`. Click **Next**.

### Step 115
Event source: **AWS events**. Under "Event pattern," choose service **Inspector2**, event type: **Inspector2 Finding**.

### Step 116
Click **Edit pattern** (or similar) and paste this exact JSON, replacing whatever is there:
```json
{
  "source": ["aws.inspector2"],
  "detail-type": ["Inspector2 Finding"],
  "detail": {
    "severity": ["CRITICAL"]
  }
}
```

### Step 117
Click **Next**. Target type: **AWS service**. Select target: **SNS topic**. Choose `inspector-lab-alerts`.

### Step 118
Click **Next**, then **Next** again, then **Create rule**.

✅ **Checkpoint (Part 6 complete):** Any new Critical finding from now on will email you.

---

# PART 7 — Cleanup (Do This Last, In This Exact Order)

> Doing this out of order can cause errors (e.g., you can't delete a role that's still attached to a running resource).

### Step 119
**EventBridge:** Go to EventBridge → Rules → select `inspector-critical-findings` → **Delete**.

### Step 120
**SNS:** Go to SNS → Topics → select `inspector-lab-alerts` → **Delete**.

### Step 121
**Lambda:** Go to Lambda → Functions → select `inspector-lab-lambda` → **Actions → Delete** → confirm.

### Step 122
**IAM (Lambda role):** Go to IAM → Roles → search `inspector-lab-lambda-role` → open it → **Delete** (top right) → type the role name to confirm.

### Step 123
**ECR:** Go to ECR → Repositories → select `inspector-lab-repo` → **Delete**. The console will warn that the repository still contains images (you pushed two — the vulnerable one and the fixed one). Check the box to **force delete** the images along with the repo, then type the repo name to confirm.

### Step 124
**EC2:** Go to EC2 → Instances → select `inspector-lab-ec2` → **Instance state → Terminate instance** → confirm.

### Step 125
Wait until the instance shows **Terminated** state (1-2 minutes).

### Step 126
**IAM (EC2 role):** Go to IAM → Roles → search `EC2-SSM-Inspector-Role` → open it → **Delete** → confirm.

### Step 127
**Inspector (optional):** If you don't plan to use Inspector for anything else right now, go to Inspector → Account management → turn off EC2/ECR/Lambda scanning to avoid any future charges on new resources you might create later.

### Step 128
**Final check:** Go through EC2, ECR, Lambda, IAM, SNS, and EventBridge consoles one more time and confirm nothing named `inspector-lab-*` remains anywhere.

✅ **Lab fully complete and cleaned up.**

---

## Quick Recap (What You Just Learned By Doing)

| Part | What You Did | What You Learned |
|---|---|---|
| 1 | Turned on Inspector | Nothing gets scanned until it's explicitly activated per resource type |
| 2 | Vulnerable EC2 → patched | EC2 scanning depends entirely on the SSM Agent being registered |
| 3 | Vulnerable Docker image → fixed | Findings are tracked per image digest, not per repo/tag |
| 4 | Vulnerable Lambda → fixed | Even small functions with old dependencies get flagged |
| 5 | Prioritized findings | Severity + internet-reachability decide what to fix first |
| 6 | Set up alerts | Critical findings can automatically notify you by email |
| 7 | Cleaned up | Real AWS labs must be torn down in dependency order to avoid errors and charges |
