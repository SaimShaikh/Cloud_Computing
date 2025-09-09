# AWS Lambda 
<img width="250" height="250" alt="image" src="https://github.com/user-attachments/assets/22585e9d-9fde-4610-b454-dd09ed1ecd0f" />

---

## Overview

AWS Lambda is a serverless computing service that lets you run code in response to events without managing servers. 
You just upload your code, and AWS automatically handles the rest, scaling as needed and only charging for the time your code runs.
Lambda is ideal for event-driven workloads, microservices, scheduled tasks, lightweight APIs, glue logic, and short-lived processing jobs.

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



