---
name: infra-aws
description: AWS architecture standards - service selection (Lambda/ECS/EC2, S3, DynamoDB, RDS, SQS/SNS/EventBridge), CDK patterns, least-privilege IAM, reference architectures, cost red flags, and a security checklist. Use when designing or reviewing AWS infrastructure, writing CDK/Terraform/SAM code, or setting up IAM policies, VPCs, or serverless APIs.
---

# Skill: AWS

## Principles
- Least privilege in IAM. Always.
- Infrastructure as code (CDK preferred, Terraform if the team already uses it).
- Never hardcode credentials. Use IAM roles + Secrets Manager.
- Design for failure: things will break, the system must recover.

---

## Services by category

### Compute
| Service | When to use it |
|----------|---------------|
| Lambda | Events, short tasks (< 15 min), scales to zero |
| ECS Fargate | Containers without managing EC2, long-running workloads |
| EC2 | Full control, stateful workloads, GPU |
| App Runner | Deploy containers without configuring ECS/ALB |

### Storage
| Service | When to use it |
|----------|---------------|
| S3 | Objects, static files, backups, data lake |
| EBS | Persistent disk for EC2 |
| EFS | Shared filesystem across instances |
| DynamoDB | Key-value / documents, massive scale, low latency |
| RDS | Relational SQL (Postgres preferred) |
| ElastiCache | In-memory cache (Redis / Memcached) |

### Networking
| Service | When to use it |
|----------|---------------|
| ALB | HTTP/HTTPS load balancer with path/host routing |
| API Gateway | Serverless REST/HTTP/WebSocket APIs |
| CloudFront | Global CDN, edge caching |
| Route 53 | DNS, health checks, failover |
| VPC | Private network, public/private subnets, security groups |

### Messaging
| Service | When to use it |
|----------|---------------|
| SQS | Message queue, decoupling, automatic retry |
| SNS | Fan-out pub/sub, notifications |
| EventBridge | Event bus, event routing rules |
| Kinesis | Real-time data streaming |

---

## CDK — base patterns (TypeScript)

```typescript
import * as cdk from 'aws-cdk-lib'
import * as lambda from 'aws-cdk-lib/aws-lambda'
import * as sqs from 'aws-cdk-lib/aws-sqs'

export class AppStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props)

    // SQS queue with DLQ
    const dlq = new sqs.Queue(this, 'DLQ', {
      retentionPeriod: cdk.Duration.days(14),
    })

    const queue = new sqs.Queue(this, 'Queue', {
      visibilityTimeout: cdk.Duration.seconds(300),
      deadLetterQueue: { queue: dlq, maxReceiveCount: 3 },
    })

    // Lambda with minimal permissions
    const fn = new lambda.Function(this, 'Handler', {
      runtime: lambda.Runtime.PYTHON_3_12,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('src'),
      environment: {
        QUEUE_URL: queue.queueUrl,
      },
    })

    queue.grantSendMessages(fn)
  }
}
```

---

## IAM — golden rules

```typescript
// ✅ Least privilege: only what it needs
fn.addToRolePolicy(new iam.PolicyStatement({
  actions: ['s3:GetObject'],
  resources: [`${bucket.bucketArn}/uploads/*`],
}))

// ❌ Never this
fn.addToRolePolicy(new iam.PolicyStatement({
  actions: ['s3:*'],
  resources: ['*'],
}))
```

- One role per service/function
- Never `*` in Actions or Resources in production
- Rotate access keys. Prefer roles over keys whenever possible
- Enable MFA on accounts with elevated permissions

---

## Common architectures

### Serverless API
```
Route 53 → CloudFront → API Gateway → Lambda → DynamoDB
                                             → RDS Proxy → RDS
```

### Containerized microservice
```
ALB → ECS Fargate → RDS (Postgres)
                 → ElastiCache (Redis)
                 → SQS (async tasks)
                      → Lambda worker
```

### Data pipeline
```
S3 (raw) → Lambda (trigger) → SQS → Lambda (process) → S3 (processed)
                                                       → DynamoDB
```

---

## Costs — red flags

- Lambda with memory > 1GB for simple tasks → reduce it or move to ECS
- RDS always on with < 10% CPU → consider Aurora Serverless v2
- S3 without lifecycle policies → data grows unchecked
- EC2 without auto-scaling → over-provisioned during off-peak hours
- No Reserved Instances/Savings Plans for predictable workloads → you're overpaying on-demand

---

## AWS security checklist

- [ ] VPC with private subnets for DB and compute
- [ ] Security groups with the minimum ports open
- [ ] Secrets in AWS Secrets Manager, not in Lambda env vars
- [ ] S3 buckets with Block Public Access enabled
- [ ] CloudTrail enabled in every region
- [ ] GuardDuty enabled
- [ ] Automatic RDS backups with retention > 7 days
- [ ] Encryption at rest on S3, RDS, EBS

---

## Common architecture decisions on AWS

Apply the decision protocol from CLAUDE.md when facing:
- **Compute:** Lambda vs ECS Fargate vs EC2
- **DB:** DynamoDB vs RDS vs Aurora Serverless
- **API:** API Gateway vs ALB vs App Runner
- **IaC:** CDK vs Terraform vs SAM
- **Messaging:** SQS vs EventBridge vs SNS
- **Cache:** ElastiCache Redis vs DynamoDB DAX vs CloudFront
