# High-Availability Trading System Architecture

**AWS Solution for 500 RPS | p99 < 100ms | Multi-AZ**

## Architecture Overview

```
Users → CloudFront → API Gateway → ECS Fargate → Aurora + DynamoDB + Redis
                                ↓
                          SQS → Lambda → Kinesis → S3
```

### Core Components

| Layer | Service | Purpose |
|-------|---------|---------|
| **Edge** | CloudFront + WAF | Global CDN, DDoS protection |
| **API** | API Gateway (REST + WebSocket) | Managed APIs, throttling, auth |
| **Compute** | ECS Fargate + Lambda | Trading engine, async processing |
| **Cache** | ElastiCache Redis | Orderbook cache (sub-ms latency) |
| **Database** | Aurora PostgreSQL + DynamoDB | Transactions + hot data |
| **Messaging** | SQS FIFO + Kinesis | Order queue, event streaming |
| **Security** | Cognito + KMS + Secrets Manager | Auth, encryption, credentials |


## Key Architecture Decisions

### 1. Why ECS Fargate over Lambda?
**Chosen**: ECS Fargate  
**Rationale**: Trading engine needs stateful connections and long-running processes (orderbook in memory). Lambda 15-min timeout insufficient. 

### 2. Why Aurora + DynamoDB hybrid?
**Chosen**: Aurora for transactions, DynamoDB for hot data  
**Rationale**: Aurora provides ACID guarantees for order matching. DynamoDB delivers single-digit ms latency for wallet balances (100K+ RPS).

### 3. Why API Gateway over ALB?
**Chosen**: API Gateway  
**Rationale**: Built-in throttling (10K RPS), native WebSocket support, Cognito integration

## Performance

### Latency Breakdown (Order Placement)
- API Gateway: 1ms (throttling)
- Cognito: 2ms (JWT validation)
- DynamoDB: 3ms (balance check)
- Redis: 2ms (orderbook fetch)
- Order matching: 10ms (in-memory)
- SQS enqueue: 5ms
- Total: ~30ms

### Throughput
- Current: 500 RPS baseline
- Peak capacity: 2,000 RPS (before major changes)
- Headroom: 4x current load

## High Availability

### Multi-AZ Deployment
```
AZ-1a: ECS tasks + Redis Primary + Aurora Writer
AZ-1b: ECS tasks + Redis Replica + Aurora Reader
AZ-1c: Aurora Reader

Automatic failover: < 1 minute
RTO: < 1 min | RPO: 0 (sync replication)
```

### Auto-Scaling
- **ECS**: CPU > 70% → add 50% tasks (60s scale-out)
- **Aurora**: Add read replicas when CPU > 75%
- **DynamoDB**: On-Demand auto-scaling

---

## Scaling Strategy

### 10x Growth (5,000 RPS)
- Scale ECS: 10 → 100 tasks
- Aurora: db.r6g.large → db.r6g.2xlarge
- Redis: Single → 3-node cluster
- SQS: 1 queue → 5 queues (partitioned by symbol)

**Cost**: $32,500/month → $24,000 (with reserved instances)

### 100x Growth (50,000 RPS)
- Multi-region deployment (us-east-1, eu-west-1, ap-southeast-1)
- Migrate SQS → MSK (Kafka) for 1M+ TPS
- Database sharding: 10 Aurora clusters
- API caching layer (80% hit ratio)

**Cost**: $283,000/month

## Cost Analysis

### Baseline (500 RPS)
| Service | Monthly Cost |
|---------|-------------:|
| API Gateway | $1,050 |
| ECS Fargate (10 tasks) | $730 |
| Aurora PostgreSQL (3 instances) | $900 |
| ElastiCache Redis | $400 |
| DynamoDB | $300 |
| Other (Lambda, SQS, S3, CloudWatch) | $450 |
| **TOTAL** | **$3,830** |

**Cost per 1,000 requests**: $7.66  
**Optimized (reserved)**: $3,064/month (20% savings)

---

## Security

### Defense-in-Depth (5 Layers)
1. **Edge**: CloudFront + WAF (DDoS, SQL injection, XSS)
2. **API**: Throttling + JWT validation
3. **Network**: Private subnets + Security Groups
4. **Application**: IAM roles + Secrets rotation
5. **Data**: KMS encryption at rest + TLS 1.3 in transit

