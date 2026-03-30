# aws-architecture-diagrams

A growing collection of AWS architecture diagrams, published at intervals .

Each diagram covers a different pattern, touches different AWS services, and includes a breakdown of how traffic flows and why each component was chosen. The series is designed to progress from foundational patterns to advanced, production-grade architectures.

---

## Diagrams

| # | Architecture | Services | Difficulty |
|---|---|---|---|
| 01 | [Production-grade serverless API](#01---production-grade-serverless-api) | Lambda, API Gateway, DynamoDB, Cognito, WAF, SES, Secrets Manager, CloudWatch | Intermediate |

---

## 01 - Production-grade serverless API

<img width="1251" height="556" alt="Screen Shot 2026-03-30 at 12 10 59 PM" src="https://github.com/user-attachments/assets/3322f50b-d5d7-4253-9379-3d9ac8dfe701" />


**Medium article:** [The One Thing Missing From Most Serverless API Diagrams](https://medium.com/@ayotomiwavictor1/the-one-thing-missing-from-most-serverless-api-diagrams-ff3bf43dd0ea)

### What this solves

A fully serverless REST API that handles authentication, security filtering, data persistence, async event processing, and observability — with no servers to manage, no VPCs to configure, and no capacity planning required.

### Traffic flow

1. **Client** sends an HTTPS request to the API
2. **AWS WAF** inspects the request first — blocks SQL injection, XSS, and bot traffic before anything else is invoked. Malicious traffic never reaches the API layer and never triggers a billable Lambda invocation
3. **Amazon Cognito** handles user authentication — user pools, sign-up, sign-in, and JWT token issuance
4. **API Gateway** receives clean, authenticated traffic, validates the JWT token, and rejects unauthenticated requests immediately before routing to the correct Lambda function
5. **Lambda functions** are invoked per route — each function owns one operation with its own least-privilege IAM role
6. **Amazon DynamoDB** stores and retrieves data — on-demand capacity handles thousands of concurrent Lambda connections natively, no connection pooling needed
7. **DynamoDB Streams** captures every data change and triggers the event processor asynchronously — keeping the write path fast
8. **Amazon SES** sends transactional emails triggered by the stream — decoupled from the main request path
9. **Secrets Manager** stores API keys and credentials — fetched by Lambda at runtime, never hardcoded
10. **Amazon CloudWatch** receives logs and metrics from every Lambda invocation — the primary observability layer when there are no servers to SSH into

### Why no VPC?

All services here are AWS-managed and communicate via IAM and service APIs, not network routing. A VPC would only be introduced if Lambda needed to access private resources such as RDS. Adding a VPC unnecessarily increases cold start latency.

### Key trade-offs

| Decision | Why |
|---|---|
| WAF before everything | Blocks malicious traffic before it triggers a billable Lambda invocation |
| Lambda per route | Fault isolation — a bug in one function cannot affect another |
| DynamoDB over RDS | Handles massive concurrent Lambda connections natively via HTTP |
| Async email via Streams | Keeps POST response time fast — email sending does not block the write path |
| Secrets Manager over env vars | No credentials exposed if function code is ever compromised |

### Scaling limits

- Lambda: 1,000 concurrent executions per region by default (raisable)
- API Gateway: 10,000 requests per second by default (raisable)
- DynamoDB on-demand: scales automatically with traffic
- Cold starts: noticeable under sudden heavy burst traffic on fresh Lambda instances

---

## About this series

I am a cloud engineer documenting my learning in public. Every diagram in this series is intentional — each one introduces services or patterns the previous one did not.



## Connect

- Medium: [medium.com/@ayotomiwavictor1](https://medium.com/@ayotomiwavictor1)
- GitHub: [github.com/Zaamaar](https://github.com/Zaamaar)
