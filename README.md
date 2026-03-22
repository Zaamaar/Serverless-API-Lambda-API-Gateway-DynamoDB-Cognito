# Serverless-API-Lambda-API-Gateway-DynamoDB-Cognito
# Serverless API Architecture — AWS Lambda, API Gateway, DynamoDB & Cognito

A production-grade serverless API architecture designed for scalable, authenticated REST APIs on AWS. No servers to manage, no capacity planning, no patching — just business logic and managed services.

This is the third architecture in my hands-on AWS cloud architecture series, following a monolithic single-tier design and a full 3-tier EC2-based architecture (deployed as a live weather app with Terraform).



## What This Architecture Solves

Traditional backend APIs require server provisioning, OS patching, autoscaling configuration, connection pool management, and custom authentication systems. This serverless architecture eliminates all of that:

- **Zero server management** — AWS manages all underlying infrastructure
- **Scales automatically** — from zero requests to millions without configuration changes
- **Pay per use** — billed per Lambda invocation, not per running server
- **Built-in authentication** — Cognito handles user sign-up, sign-in, and JWT token management
- **Edge security** — WAF blocks malicious traffic before it reaches any compute or data layer

---

## Architecture Overview

The architecture is organised into three layers inside the AWS Cloud boundary:

```
User (mobile app / browser)
        ↓
┌─────────────────────────────────────┐
│         EDGE / AUTH LAYER           │
│  Cognito → WAF → API Gateway        │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│        APPLICATION LAYER            │
│  Lambda (x4) + IAM Roles            │
│  + CloudWatch Logs                  │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│           DATA LAYER                │
│         DynamoDB                    │
└─────────────────────────────────────┘
```

---

## Components

### Edge / Auth Layer

| Service | Role | Why It Exists |
|---|---|---|
| Amazon Cognito | User authentication | Manages sign-up, sign-in, JWT token issuance — replaces weeks of custom auth development |
| AWS WAF | Web Application Firewall | Blocks SQL injection, XSS, bots before requests reach API Gateway or Lambda |
| Amazon API Gateway | API entry point | Validates JWT tokens, routes requests to correct Lambda, applies rate limiting |

### Application Layer

| Service | Role | Why It Exists |
|---|---|---|
| AWS Lambda (x4) | Business logic | Stateless functions that run only when triggered — zero cost at idle |
| IAM Execution Roles | Least privilege permissions | Each Lambda has scoped permissions — compromised function cannot access unrelated resources |
| Amazon CloudWatch | Observability | Logs every invocation — execution time, errors, memory usage |

### Data Layer

| Service | Role | Why It Exists |
|---|---|---|
| Amazon DynamoDB | NoSQL database | HTTP-based API handles thousands of concurrent Lambda connections natively — no connection limit issues |

---

## Traffic Flow

### Authentication Flow
```
User enters credentials
    → Cognito validates password
    → Cognito issues JWT access token
    → Token stored on user device
```

### API Request Flow
```
User sends HTTP request + JWT token
    → WAF inspects for malicious patterns → blocks if unsafe
    → API Gateway validates JWT with Cognito → rejects if invalid/expired
    → API Gateway routes to correct Lambda (path + HTTP method)
    → Lambda executes business logic using IAM execution role
    → Lambda reads/writes DynamoDB
    → Response returned through API Gateway to user
    → CloudWatch logs the entire invocation
```

---

## Lambda Functions

| Function | Route | DynamoDB Permission | Table |
|---|---|---|---|
| Get Users | GET /users | GetItem, Query, Scan | users |
| Create Item | POST /item | PutItem | items |
| Delete Item | DELETE /item | DeleteItem, GetItem | items |
| Get Orders | GET /orders | GetItem, Query | orders |

Each function has its own IAM execution role following least privilege — a function that reads users cannot write or delete. A function that manages items cannot access orders.

---

## IAM Security Model

Every Lambda function receives:

**Baseline (all functions):**
```json
{
  "Effect": "Allow",
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents"
  ],
  "Resource": "*"
}
```

**Per function (example — GET /users):**
```json
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:GetItem",
    "dynamodb:Query"
  ],
  "Resource": "arn:aws:dynamodb:us-east-1:ACCOUNT_ID:table/users"
}
```

No function has access to tables it does not need. No function has write access if it only reads.

---

## Why DynamoDB Over RDS

| Concern | RDS MySQL | DynamoDB |
|---|---|---|
| Connection model | Persistent TCP connections (limited pool) | HTTP API (no connection limit) |
| Concurrent Lambda connections | Problematic at scale — connection exhaustion | Native — thousands of concurrent calls |
| Management | Patches, backups, parameter groups | Fully managed |
| Query model | Complex SQL joins | Key-value and document |
| Cost at low traffic | Fixed hourly rate | Pay per request |
| Best for | Complex relational queries, transactions | High-throughput, simple access patterns |

For a serverless API with simple CRUD operations and unpredictable traffic, DynamoDB is the natural choice. RDS is better suited for complex relational data and when the EC2-based compute layer handles connection pooling.

---

## Comparison With EC2-Based Architecture

This architecture is the deliberate serverless counterpart to my ZamWeather 3-tier EC2 architecture.

| Aspect | EC2 3-Tier (ZamWeather) | Serverless API |
|---|---|---|
| Compute | EC2 t2.micro + ASG | Lambda functions |
| Database | RDS MySQL | DynamoDB |
| Auth | Custom / none | Cognito |
| Networking | VPC, subnets, NAT Gateways | Not required |
| Scaling | Minutes (new EC2 instance) | Milliseconds |
| Cost model | Always-on hourly rate | Per invocation |
| Ops overhead | High (patching, SSH, user data) | Near zero |
| Best for | Sustained traffic, full control | Event-driven, variable traffic |

Neither is universally better. The right choice depends on traffic patterns, team skills, query complexity, and operational capacity.

---

## Production Considerations

- **Cognito SLA is 99.9%** — your API availability is bounded by its weakest dependency. Cache JWT validation where possible to reduce Cognito dependency on every request
- **Lambda cold starts** — first invocation after idle period takes 100ms-3s. Use provisioned concurrency for latency-sensitive endpoints
- **DynamoDB capacity modes** — on-demand mode for unpredictable traffic, provisioned mode for predictable traffic at lower cost
- **WAF costs** — approximately $5/month per Web ACL plus $1/million requests. Factor into cost estimates
- **API Gateway throttling** — configure per-route throttle limits to prevent one endpoint from consuming all capacity

---

## What Is Next

This is an architecture diagram — the implementation is the next step:

- Deploy using AWS SAM (Serverless Application Model) or Terraform
- Write Lambda function code
- Configure Cognito user pools and app clients
- Set up DynamoDB tables with appropriate partition keys
- Wire API Gateway routes to Lambda functions
- Write a Medium article documenting the full deployment

---

## Related Articles

- **This architecture — design breakdown:** [paste Medium link]
- **ZamWeather — live 3-tier deployment:** https://medium.com/@ayotomiwavictor1/building-a-secure-globally-accelerated-3-tier-web-architecture-on-aws-49b23c180173
- **3-tier architecture design:** https://medium.com/@ayotomiwavictor1/building-a-secure-globally-accelerated-3-tier-web-architecture-on-aws-49b23c180173

## Related Repositories

- **ZamWeather (live deployed app):** https://github.com/Zaamaar/zamweather

---

*Part of an ongoing hands-on AWS cloud architecture series. Diagrams created with diagrams.net (draw.io) using AWS architecture icons.*

*Built towards becoming a cloud architect — designing, deploying, and documenting real AWS architectures.*
