# Simple Architecture Diagram

```
┌───────────────────────────────────────────────────────────────┐
│                         USERS                                 │
│                    (Web + Mobile)                             │
└──────────────────────────┬────────────────────────────────────┘
                           │
┌──────────────────────────▼────────────────────────────────────┐
│                    EDGE LAYER                                 │
│                                                               │
│  CloudFront CDN → AWS WAF → Route53 DNS                      │
│  (Global distribution, DDoS protection)                       │
└──────────────────────────┬────────────────────────────────────┘
                           │
┌──────────────────────────▼────────────────────────────────────┐
│                    API LAYER                                  │
│                                                               │
│  API Gateway (REST + WebSocket)                              │
│  - Throttling: 10,000 RPS                                    │
│  - Auth: Cognito JWT validation                              │
└──────────────────────────┬────────────────────────────────────┘
                           │
┌──────────────────────────▼────────────────────────────────────┐
│               COMPUTE LAYER (Multi-AZ)                        │
│                                                               │
│  ┌─────────────────┐         ┌──────────────────┐           │
│  │  ECS Fargate    │         │  AWS Lambda      │           │
│  │  Trading Engine │◀────────│ Order Processing │           │
│  │  (10 tasks)     │         │                  │           │
│  └────────┬────────┘         └────────┬─────────┘           │
│           │                           │                      │
│           │      ┌───────────────────┐│                      │
│           └─────▶│  SQS FIFO Queue   ││                      │
│                  │  (Exactly-once)   ││                      │
│                  └───────────────────┘│                      │
└────────────┬──────────────────────────┴───────────────────────┘
             │
┌────────────▼──────────────────────────────────────────────────┐
│                  CACHE LAYER (Multi-AZ)                       │
│                                                               │
│  ElastiCache Redis (Primary + Replica)                       │
│  - Orderbook cache (sub-ms latency)                          │
│  - Automatic failover                                        │
└────────────┬──────────────────────────────────────────────────┘
             │
┌────────────▼──────────────────────────────────────────────────┐
│               DATA LAYER (Multi-AZ)                           │
│                                                               │
│  ┌──────────────────┐       ┌───────────────────┐           │
│  │ Aurora PostgreSQL│       │    DynamoDB       │           │
│  │ Writer + Readers │       │ User Sessions     │           │
│  │ (Transactions)   │       │ & Wallets         │           │
│  └────────┬─────────┘       └─────────┬─────────┘           │
│           │                           │                      │
│           ▼                           ▼                      │
│  ┌──────────────────┐       ┌───────────────────┐           │
│  │ Kinesis Streams  │──────▶│   S3 Buckets      │           │
│  │ Trade Events     │       │ Historical Data   │           │
│  └──────────────────┘       └───────────────────┘           │
└───────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│              OBSERVABILITY & SECURITY                         │
│                                                               │
│  CloudWatch (Metrics + Logs) | X-Ray (Tracing)              │
│  Secrets Manager | KMS Encryption | IAM Roles                │
└───────────────────────────────────────────────────────────────┘
```

---

## Request Flow: Order Placement

```
1. User → CloudFront → API Gateway
   └─ Validate JWT (Cognito): 2ms

2. API Gateway → ECS Trading Engine
   └─ Check balance (DynamoDB): 3ms
   └─ Fetch orderbook (Redis): 2ms
   └─ Match order (in-memory): 10ms

3. ECS → SQS FIFO Queue
   └─ Enqueue order: 5ms
   └─ Return HTTP 202 Accepted

   TOTAL: ~30ms

4. ASYNC: SQS → Lambda
   └─ Persist to Aurora
   └─ Update wallet (DynamoDB)
   └─ Publish to Kinesis → S3
```

---

## Multi-AZ High Availability

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  AZ us-east-1a  │    │  AZ us-east-1b  │    │  AZ us-east-1c  │
├─────────────────┤    ├─────────────────┤    ├─────────────────┤
│                 │    │                 │    │                 │
│ 5 ECS Tasks     │    │ 5 ECS Tasks     │    │                 │
│                 │    │                 │    │                 │
│ Redis Primary   │◀──▶│ Redis Replica   │    │                 │
│                 │    │                 │    │                 │
│ Aurora Writer   │───▶│ Aurora Reader   │    │ Aurora Reader   │
│                 │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘

If AZ-1a fails → Automatic failover < 1 minute
                 All traffic routes to AZ-1b
                 Zero data loss (sync replication)
```

---

## Scaling Path

```
CURRENT (500 RPS)
├─ ECS: 10 tasks
├─ Aurora: 1 writer + 2 readers
├─ Redis: Primary + Replica
└─ Cost: $3,830/month

      ↓ 10x Traffic

5,000 RPS
├─ ECS: 100 tasks
├─ Aurora: 1 writer + 5 readers
├─ Redis: 3-node cluster
├─ SQS: 5 partitioned queues
└─ Cost: $24,000/month

      ↓ 10x Traffic

50,000 RPS
├─ Multi-region (3 regions)
├─ MSK (Kafka) instead of SQS
├─ Database sharding (10 clusters)
├─ API caching layer
└─ Cost: $180,000/month

---

## Key Metrics

**Performance**
- Latency (p99): 30ms
- Throughput: 500 RPS baseline
- Availability: 99.95% SLA

**Costs**
- Baseline: $3,830/month
- Per 1,000 requests: $7.66
- Optimized: $2,800/month (with reserved)

**Recovery**
- AZ failure: < 1 min (RTO)
- Data loss: 0 (RPO)
- Backup retention: 35 days

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| CDN | CloudFront + WAF |
| API | API Gateway (REST + WS) |
| Auth | Cognito + JWT |
| Compute | ECS Fargate + Lambda |
| Queue | SQS FIFO |
| Cache | ElastiCache Redis |
| Database | Aurora PostgreSQL |
| NoSQL | DynamoDB |
| Streaming | Kinesis + S3 |
| Monitoring | CloudWatch + X-Ray |
| Security | KMS + Secrets Manager |
