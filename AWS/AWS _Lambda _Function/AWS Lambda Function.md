# AWS Lambda 
<img width="250" height="250" alt="image" src="https://github.com/user-attachments/assets/22585e9d-9fde-4610-b454-dd09ed1ecd0f" />

---

## Overview

AWS Lambda is a serverless computing service that lets you run code in response to events without managing servers. 
You just upload your code, and AWS automatically handles the rest, scaling as needed and only charging for the time your code runs.
Lambda is ideal for event-driven workloads, microservices, scheduled tasks, lightweight APIs, and short-lived processing jobs.

* Why is Lambda called serverless?
AWS Lambda is called “serverless” because you don’t manage servers — AWS does.
But don’t get it twisted — servers still exist.
They’re just hidden from you. 😏


* ⚙️ What Happens Under the Hood

- When you deploy your function to Lambda:

- AWS keeps a pool of compute resources (actual servers under the hood).

- When your function is triggered — AWS automatically provisions one of those servers.

- Your code runs there for a few milliseconds or seconds.

- When done — the server is destroyed or put back in the pool.

- You pay only for the time your code actually ran (not for the idle time).

So yeah — there are servers, but you never see, configure, or maintain them.
Hence, “Serverless.”
---

### AWS Lambda language support 
- Node.js (JavaScript)
- Python
- Java
- C# (.NET Core)
- Ruby
- Custom Runtime API

---

## When to use Lambda

* Event-driven processing: file uploads, DB streams, message queues, scheduled jobs.
* Autoscaling without managing servers.
* Pay-per-invocation cost model.
* Image Processing:Let’s say users upload images to your app. You can set up Lambda to automatically resize, compress, or even apply filters to each image as it’s uploaded.
  Avoid Lambda for: long-running stateful processes, apps that require persistent local state, very high memory/CPU requirements where dedicated instances perform better.
* Data Transformation: If you need to clean up or process data before storing it in a database, a Lambda function can handle that transformation automatically.
* Real-Time Notifications: If an event happens—like a new usersigning up—you can use Lambda to trigger an email, SMS, orother notifications instantly.
---

## Key Concepts

* **Function** — your code + configuration.
* **Handler** — the entrypoint that Lambda invokes.
* **Runtime** — Node.js, Python, Java, Go, .NET, Ruby, or custom runtime/container.
* **Event** — payload that triggers the function (API call, S3 PUT, DB stream, etc.).
* **Concurrency** — number of simultaneous executions.
* **Provisioned Concurrency** — keeps function initialized for consistent cold-start latency.
* **Layers** — share common libraries across functions.
* **Cold start** — initial startup penalty for a function container.

---

## Architecture Patterns & Event Sources

Common patterns:

* **API backends**: API Gateway → Lambda (proxy integration).
* **Data pipelines**: S3 events → Lambda → transform → S3/DynamoDB.
* **Stream processing**: Kinesis / DynamoDB Streams → Lambda.
* **Asynchronous decoupling**: SNS/SQS → Lambda.
* **Scheduled jobs**: EventBridge / CloudWatch Events → Lambda.

Event sources include: API Gateway, Application Load Balancer (ALB), S3, DynamoDB Streams, Kinesis Streams, SNS, SQS, EventBridge, CloudWatch Logs subscription.

---


---

## 🕰️ Before Lambda — The Traditional Way

Before AWS Lambda arrived, developers had to **manually manage servers** to run code. Here’s what that looked like:

### You had to:

* Buy or rent servers (EC2, on-premise, etc.)
* Install OS, runtime, and dependencies
* Scale manually — more traffic = more servers
* Keep servers running 24/7 even during idle times
* Handle patching, monitoring, logging, and maintenance

### Example

If you wanted to process image uploads:

1. Launch an EC2 instance
2. Write and deploy the image-processing app
3. Keep the instance running continuously — you pay even when idle

### Common technologies used before

* EC2 / On-prem servers
* Docker containers
* Auto Scaling groups
* Load balancers

All of these required significant infrastructure management — aka "infrastructure babysitting." 👶

---

## ⚙️ After Lambda — The Serverless Way

AWS Lambda changed the game. Instead of managing servers, you focus on small, event-driven functions.

### Now you:

* Write only the function (business logic)
* Deploy it to AWS Lambda
* Trigger execution with events (S3, API Gateway, CloudWatch, SNS, etc.)
* Pay only while the function runs — no idle billing
* Offload scaling, patching, and server maintenance to AWS

### Example (same image processing case)

Image upload → triggers Lambda → processes image → done. Compute disappears after the function completes, so you don’t pay for idle time.

---

## 🔄 Before vs After Lambda — Quick Comparison

| Feature               | Before Lambda (Traditional) | After Lambda (Serverless) |
| --------------------- | --------------------------: | ------------------------: |
| **Server Management** |          You manage servers |    AWS manages everything |
| **Billing**           |              Pay for uptime | Pay per request & runtime |
| **Scalability**       |      Manual or auto scaling |       Auto-scale built-in |
| **Deployment**        |            Deploy whole app |    Deploy single function |
| **Maintenance**       |        OS updates, patching |     None (AWS handles it) |
| **Startup Time**      |              Always running |    Cold start (on-demand) |
| **Use Case Fit**      |           Long-running apps |         Event-driven apps |

---

## TL;DR

* **Before Lambda:** You managed servers and infrastructure. More overhead, more cost during idle time.
* **After Lambda:** You manage code only. AWS handles scaling and maintenance, billing is per invocation.


---


---

### what is event
In AWS Lambda (and event-driven systems in general), an event is basically a data record that describes something that just happened.

* 👉 Think of it like a message or a trigger signal. It’s JSON-formatted in AWS and passed into your Lambda function as the event parameter.

* In plain words: Event = "Hey, something happened — here are the details."
* It tells your Lambda what happened, where, and gives the context/data you need to act on it.

---

### Event-driven execution
* Event-driven Lambda = Lambda functions invoked automatically when something happens. That “something” — an event — comes from AWS services (S3 upload, SQS message, API call, DynamoDB stream, etc.). You write the function, wire a trigger, and AWS runs it when the event arrives. Classic pub/sub vibes, but serverless.

### Typical flow (example: S3 → Lambda)

- File uploaded to S3 bucket.

- S3 emits s3:ObjectCreated:* event.

- S3 triggers the Lambda (async).

- Lambda handler gets an event JSON listing bucket + object key.

Lambda processes the file (validate, transform, index, etc.), writes result/metadata somewhere.

---
### Lambda Disadvantage or Limitations 
* Deployment Package Limits :
   - Direct upload (zip) → 50 MB.
   - Unzipped code (in Lambda) → 250 MB.
   - Container image → up to 10 GB (pushed via ECR).
* Short-running tasks (typical invocation < 15 minutes): max 15 minutes per invocation. (If your job takes longer, Lambda isn’t your tool — use ECS, Batch, or Step Functions.)
* Stateless: Lambda functions don’t keep state between invocations, so they’re best for tasks that don’t require long-term memory.
* Cold Start Delays: If a Lambda function hasn’t run in a while, there’s sometimes a slight delay—called a 'cold start'—when itstarts up. This can add a little latency, but AWS provides ways to mitigate it for critical functions.



### ⚙️ What is a Lambda Layer?

* A Lambda Layer is basically a shared package of code or dependencies that you can attach to multiple Lambda functions. It’s AWS’s way of letting you reuse libraries or config files without bloating every function’s ZIP.

* Think of it like this:

> 🎒 “A Layer is a backpack that your Lambda function can wear — full of tools (libraries, configs, binaries) it needs.”
---

### 💰 AWS Lambda Pricing Explained (with Examples)

* AWS Lambda pricing depends on a few simple factors — understand these and you won’t get surprised by the bill.

1️⃣ Number of Invocations (Requests)

- Each function execution = 1 invocation.

- First 1,000,000 requests/month are free.

- After that, it's roughly $0.20 per 1M requests (varies by region).

- Example: If your function runs 10M times in a month → you pay for 9M requests (since 1M are free) → ~ $1.80.

2️⃣ Duration (Execution Time)

- Billed for how long the function runs (from start to finish), measured in milliseconds (100ms granularity).

- Shorter run time = cheaper.

- Example: Function runs 500ms and is called 1M times → you pay for that total execution time.

3️⃣ Memory Allocation (RAM)

- You select memory from 128 MB to 10,240 MB.

- More memory → more CPU allocation and higher cost per millisecond.

- Pricing scales linearly with memory: 512 MB costs 4× 128 MB for same duration.

4️⃣ Additional Features

- Provisioned Concurrency: Keeps instances warm for lower latency — adds a per-hour and per-invocation cost.

- Data transfer (egress): Standard AWS data transfer charges apply when Lambda sends data out.

- CloudWatch Logs: Lambda writes logs to CloudWatch; excessive logging increases cost.

* ⚙️ Simplified Formula
Total Cost = (Number of Requests × Request Price)
+ (Duration in seconds × Memory (GB) × GB-second Price)
+ Additional feature costs (Provisioned Concurrency, Data Transfer, Logs)

### Quick example calculation (approximate):
1M requests × $0.20 per million = $0.20
+ Execution: 128 MB × 100 ms × 1M invocations -> counts toward GB-seconds free tier
= ~ $0.20 total (very cheap)
---

## Troubleshooting & FAQ

**Q: My function times out**
A: Increase the timeout in configuration, or optimize your code. Use X-Ray to find slow parts.

**Q: Why am I seeing `AccessDenied` when Lambda tries to access S3?**
A: Verify the function's execution role has the correct S3 permissions and the resource ARN is correct.

**Q: How to reduce cold starts?**
A: Reduce package size, prefer lighter runtimes (e.g., Node/Python), initialize heavy objects lazily, or enable Provisioned Concurrency.

**Q: How to call other AWS services securely?**
A: Use IAM roles attached to Lambda and grant minimal permissions. Avoid long-lived credentials in code.

---



