# AWS Lambda — Complete README

> Concise, practical, and ready-to-use documentation for AWS Lambda. Covers concepts, example handlers (Python & Node.js), deployment options, permissions, monitoring, best practices, cost notes, and troubleshooting.

---

## Table of Contents

1. [Overview](#overview)
2. [When to use Lambda](#when-to-use-lambda)
3. [Key Concepts](#key-concepts)
4. [Architecture Patterns & Event Sources](#architecture-patterns--event-sources)
5. [Quick Start — Hello World (Python & Node.js)](#quick-start--hello-world-python--nodejs)
6. [Deployment Options](#deployment-options)
7. [Permissions & IAM](#permissions--iam)
8. [Environment Variables & Config](#environment-variables--config)
9. [Monitoring, Logging & Tracing](#monitoring-logging--tracing)
10. [Local Development & Testing](#local-development--testing)
11. [CI/CD Example (GitHub Actions)](#cicd-example-github-actions)
12. [Best Practices](#best-practices)
13. [Security Considerations](#security-considerations)
14. [Cost Considerations](#cost-considerations)
15. [Troubleshooting & FAQ](#troubleshooting--faq)
16. [References & Further Reading](#references--further-reading)
17. [License](#license)

---

## Overview

AWS Lambda is a serverless compute service that runs your code in response to events and manages the underlying compute resources automatically. You write functions, upload them (or connect via container image), and Lambda executes them on demand.

Lambda is ideal for event-driven workloads, microservices, scheduled tasks, lightweight APIs, glue logic, and short-lived processing jobs.

---

## When to use Lambda

* Short-running tasks (typical invocation < 15 minutes).
* Event-driven processing: file uploads, DB streams, message queues, scheduled jobs.
* Autoscaling without managing servers.
* Pay-per-invocation cost model.

Avoid Lambda for: long-running stateful processes, apps that require persistent local state, very high memory/CPU requirements where dedicated instances perform better.

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

## Quick Start — Hello World (Python & Node.js)

### Python (3.11+) — handler: `lambda_function.py`

```python
# lambda_function.py
import os
import json

def lambda_handler(event, context):
    # simple echo function
    name = event.get('name', 'World')
    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': f'Hello, {name}!',
            'region': os.environ.get('AWS_REGION')
        })
    }
```

### Node.js (18.x) — handler: `index.js`

```javascript
// index.js
exports.handler = async (event) => {
  const name = event?.name || 'World'
  return {
    statusCode: 200,
    body: JSON.stringify({ message: `Hello, ${name}!` })
  }
}
```

**Deploy quickly via AWS Console**: create function → choose runtime → paste code → configure trigger (API Gateway) → test.

---

## Deployment Options

1. **AWS Console** — quick for manual testing or demos. Not recommended for production deployments due to lack of repeatability.
2. **AWS CLI** — `aws lambda create-function` / `update-function-code`. Good for scripts.
3. **AWS SAM (Serverless Application Model)** — recommended for structured serverless apps with templates.
4. **Serverless Framework** — great DX for multi-service projects and plugins.
5. **Terraform** — infra-as-code that many teams prefer for multi-cloud infra.
6. **Container images** — package functions as OCI images up to 10 GB. Good if you need native dependencies.

### Example SAM `template.yaml` (minimal)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  HelloFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: lambda_function.lambda_handler
      Runtime: python3.11
      CodeUri: ./src
      MemorySize: 128
      Timeout: 10
      Events:
        Api:
          Type: Api
          Properties:
            Path: /hello
            Method: post
```

---

## Permissions & IAM

* Lambda needs permission to access other AWS services. Grant least privilege.
* Attach an IAM role (execution role) to the function. Typical managed policy: `AWSLambdaBasicExecutionRole` (for CloudWatch logs).

**Example minimal IAM policy** for reading from S3 and writing logs:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

**Notes:**

* Prefer resource-scoped ARNs instead of `"Resource": "*"` when possible.
* Use IAM Conditions to restrict by VPC or source when applicable.

---

## Environment Variables & Config

* Store non-sensitive configuration in Lambda environment variables.
* For secrets, use **AWS Secrets Manager** or **AWS Systems Manager Parameter Store (SecureString)** and grant IAM permission to access them.
* Avoid embedding secrets in code or environment variables when not encrypted.

Access environment variables in code (Python): `os.environ['MY_VAR']`.

---

## Monitoring, Logging & Tracing

* **CloudWatch Logs**: Lambda writes logs to CloudWatch automatically (via execution role). Use structured JSON logs for easier querying.
* **CloudWatch Metrics**: Invocations, Duration, Errors, Throttles, ConcurrentExecutions.
* **X-Ray**: Enable active tracing on the function to get distributed traces and to diagnose latency issues.
* **Log Insights**: Use CloudWatch Log Insights to query logs and build dashboards.

**Basic example (Python) — structured logging**

```python
import logging
import json
logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    logger.info(json.dumps({"event": event, "requestId": context.aws_request_id}))
    # ...
```

---

## Local Development & Testing

* **AWS SAM CLI**: `sam local invoke`, `sam local start-api` (emulates API Gateway).
* **LocalStack**: Full local AWS stack emulation (S3, SQS, DynamoDB, Lambda, etc.).
* **Unit tests**: mock `context` and AWS SDK calls (e.g., using `moto` for Python).
* **Integration tests**: run against a test account or LocalStack.

---

## CI/CD Example (GitHub Actions)

A minimal workflow that packages and deploys using SAM CLI:

```yaml
name: CI/CD
on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install SAM
        run: |
          pip install aws-sam-cli
      - name: Build
        run: sam build
      - name: Deploy
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_DEFAULT_REGION: us-east-1
        run: sam deploy --stack-name my-lambda-stack --no-confirm-changeset --capabilities CAPABILITY_IAM
```

---

## Best Practices

* Keep functions single-purpose and small.
* Use environment variables & parameter store for configuration.
* Prefer async event-driven designs (SQS, SNS, EventBridge) for retries and decoupling.
* Control concurrency limits to avoid downstream throttling.
* Use Layers for shared dependencies to keep deployment artifacts small.
* Minimise cold starts: reduce package size, use provisioned concurrency for latency-sensitive endpoints.
* Use connection pooling or multi-instancing strategies for database connections.
* Automate deployments with IaC (SAM/Terraform/Serverless Framework).

---

## Security Considerations

* Follow least privilege for IAM roles.
* Use Secrets Manager or Parameter Store for secrets.
* Keep dependencies up to date and scan for vulnerabilities.
* If using VPC access for the function, be careful about NAT gateway costs and cold starts.
* Restrict event source permissions and validate all incoming event payloads.

---

## Cost Considerations

* Lambda cost = (GB-seconds used) + number of requests.
* Provisioned Concurrency incurs hourly charges.
* Be mindful of high-frequency invocations and long durations — these add up.

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



